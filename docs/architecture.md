# Architecture de Rio — notes de fork

> Tracé à la main sur la base `v0.4.8` (workspace `raphamorim/rio`). Les
> références `fichier:ligne` sont indicatives et peuvent glisser au fil des
> rebases sur `main`. Pour le workflow git du fork et la map résumée des
> crates, voir [`CLAUDE.md`](../CLAUDE.md).

## 1. Le workspace en un coup d'œil

Rio est un **workspace Cargo** : un seul binaire (`frontends/rioterm`) qui
pilote une constellation de libs. Certaines sont des crates maison
réutilisables, d'autres des **forks de crates tierces** vendorisées pour
absorber des patches sans attendre l'upstream.

### Forks de crates externes (dette de maintenance)

| Crate | Forké de | Ce que fait l'original | Pourquoi le fork |
|---|---|---|---|
| `rio-window` | [`winit`](https://github.com/rust-windowing/winit) | Fenêtrage & event loop cross-platform | Patcher les spécificités terminal sans attendre upstream |
| `corcovado` | [`mio`](https://github.com/tokio-rs/mio) **0.6.x** | IO non-bloquante bas niveau (`Poll` epoll/kqueue/IOCP) | Fork maintenu de mio 0.6.x + `mio-signal-hook`/`mio-extras`, edition 2021, API Windows 11 |
| `rio-grapheme-width` | `wezterm-char-props` (WezTerm) | Tables de largeur des graphèmes (emoji, variation-sequences) | Figer/maintenir ces tables pour Rio |

### Crates maison de Rio (pas des forks)

| Crate | Rôle |
|---|---|
| `rio-backend` | Cœur non-UI : parsing ANSI (`ansi/`), modèle de grille (`crosswords/`), `performer/` (applique les séquences), config, events, sélection, clipboard, graphics (sixel/kitty) |
| `sugarloaf` | Moteur de rendu **GPU** (WebGPU), desktop + WASM. Dessine grille, onglets, curseurs |
| `teletypewriter` | Création du **pty/tty** : spawn du shell, branchement stdin/stdout |
| `rio-notifier` | Notifications OS. **Pas un fork** : macOS via `UserNotifications`/objc, Linux via `zbus` (D-Bus) |

En marge : `frontends/wasm` est l'autre frontend (cible web via sugarloaf/WASM).

Les trois pièces à retenir pour lire le reste :

- **`crosswords`** (`rio-backend/src/crosswords/`) = *le modèle d'état*. La
  grille de cellules, indépendante de l'affichage. Le nombre de lignes PTY vit
  ici.
- **`ContextManager` / `ContextGrid`** (`frontends/rioterm/src/{context,layout}/`)
  = *les onglets et les splits*. Un onglet contient une grille de panneaux
  (`ContextGrid`) ; chaque panneau est un `context` (un pty + sa grille) avec un
  `layout_rect` `[x, y, w, h]` en pixels physiques absolus.
- **L'"Island"** (`frontends/rioterm/src/renderer/island.rs`) = *la barre
  d'onglets rendue par Rio* (mode `Tab`), par opposition au `NativeTab` (barre
  OS).

Point d'architecture piège : **le rendu est gated par un flag `dirty`**.
`render()` ne présente à l'écran que si quelque chose a marqué l'état sale
(`mark_dirty` / `request_overlay_redraw`).

## 2. Démarrage — `frontends/rioterm/src/main.rs`

```
main()  frontends/rioterm/src/main.rs
  ├─ cli.rs            parse les args (--working-dir, -e cmd…)
  ├─ config_dir_path() résout ~/.config/rio ou %USERPROFILE%\AppData\Local\rio
  │  + create_config_file()  crée config.toml au 1er lancement
  ├─ setup_environment_variables()  TERM=xterm-rio, COLORTERM=truecolor, TERM_PROGRAM=rio
  ├─ EventLoop::<EventPayload>::with_user_event().build()   ← rio-window (winit)
  └─ Application::new(...).run_app()      main.rs:107 → application.rs
```

`Application::new` monte le **Router** (une `Route` par fenêtre), et la 1re
fenêtre déclenche la création d'un **Screen → ContextManager → Context**.
Créer ce premier `Context`, c'est :

```
teletypewriter::spawn   → fork le shell, ouvre le pty
crosswords::Crosswords  → alloue la grille de cellules vide
performer::Machine      → un thread lecteur par pty (voir Boucle B)
```

## 3. Régime permanent — deux boucles concurrentes

Point clé du modèle : **Rio n'est pas mono-thread**. Il y a la boucle d'events
UI (thread principal) et un thread lecteur de pty **par panneau**.

```
        ┌─────────────────────── BOUCLE A : UI (thread principal) ───────────────────────┐
        │  rio-window (winit) → application.rs : impl ApplicationHandler                  │
        │                                                                                 │
        │   window_event()   clavier / souris / resize / close / RedrawRequested          │
        │   user_event()     RioEvent::{PrepareRender, Title, PtyWrite, ClipboardLoad…}   │
        │   about_to_wait()  scheduler.rs (timers : blink curseur, throttle render)       │
        └───────────────▲───────────────────────────────────────────────┬────────────────┘
                        │ EventProxy réveille la boucle                   │ render()
             RioEvent::PrepareRender                                      ▼
                        │                              screen::render()  screen/mod.rs:3524
        ┌───────────────┴──────────── BOUCLE B ────────────┐    ├─ construit la grille + l'Island (onglets)
        │  performer::Machine (1 thread / pane)            │    │  + overlays (palette, search, scrollbar)
        │  performer/mod.rs                                │    └─ sugarloaf → GPU (present, gated par `dirty`)
        │                                                  │
        │   corcovado::Poll   attend que le pty soit lisible│
        │   pty_read()        lit ≤ N octets (lock grille)  │
        │   parser/ + handler.rs  décode l'ANSI/VT          │
        │   → mute crosswords (la grille) + mark dirty      │
        └──────────────────────────────────────────────────┘
```

Détail de coalescing (`performer/mod.rs:221`) : le thread pty ne renvoie un
`PrepareRender` **que si aucun n'est déjà en vol** — pour ne pas noyer la
boucle UI quand le shell crache 10 000 lignes.

## 4. Un aller-retour concret : tu tapes `ls⏎`

```
1. Frappe clavier
   rio-window → application.window_event(KeyboardInput)
       └─ bindings/mod.rs : la touche matche-t-elle une Action ?
            • OUI (ex. Ctrl+Shift+D, ou selectsplitleft) → screen/mod.rs exécute l'Action
            • NON (touche "texte")  → messenger.rs → écrit les octets dans le pty (teletypewriter)

2. Le shell reçoit "ls\n", s'exécute, écrit son output sur le pty

3. BOUCLE B se réveille
   corcovado::Poll signale "pty lisible"
       → Machine::pty_read lit les octets
       → performer/parser + handler.rs décodent (texte, couleurs SGR, curseur…)
       → écrivent dans crosswords (la grille)
       → EventProxy envoie RioEvent::PrepareRender à BOUCLE A

4. BOUCLE A traite user_event(PrepareRender) → programme un redraw

5. screen::render() reconstruit la scène → sugarloaf → GPU → tu vois la sortie de `ls`
```

Les Actions non-texte (étape 1, branche OUI) restent **dans la boucle A** :
`select_split`, close tab, copy/paste (`clipboard.rs`), recherche
(`hints.rs`), ouverture de split — elles mutent le `ContextManager`/`Screen` et
marquent `dirty`, sans jamais toucher au shell.

## 5. Carte des zones stimulées

| Ce que tu fais | Code stimulé (chemin chaud) |
|---|---|
| **Lancer** rio | `main.rs` → `cli.rs` → `config/` → `Application::new` → `router/` → `teletypewriter` |
| Le shell **affiche** du texte | `corcovado` (poll) → `performer/` (parse ANSI) → `crosswords` → `sugarloaf` (GPU) |
| **Taper** du texte normal | `rio-window` → `application.window_event` → `bindings/` → `messenger` → `teletypewriter` |
| **Raccourci** (split, tab, copy) | `bindings/` → `screen/mod.rs` (Action) → `layout/` ou `context/` → `mark_dirty` |
| **Naviguer** entre splits (patch fork) | `bindings/` → `screen/mod.rs:1263` → `layout::pick_directional` |
| **Redimensionner** la fenêtre | `application.window_event(Resized)` → `screen` resize → recompute lignes/colonnes + `crosswords` + Taffy |
| **Barre d'onglets** (patches fork) | `renderer/island.rs` — numéro d'onglet, snap pixels, repaint |
| **Recharger** la config à chaud | `watcher.rs` → `PrepareUpdateConfig` → réinjection dans `screen`/`renderer` |
| **Curseur qui clignote** | `scheduler.rs` (timer) → `about_to_wait` → redraw |

## 6. Résumé en une phrase

Deux boucles se parlent via `EventProxy` — le thread pty transforme des
**octets → grille** (`performer` + `crosswords`), la boucle UI transforme des
**events → Actions ou octets** (`bindings`), et `render()` transforme la
**grille → pixels** (`sugarloaf`) uniquement quand le flag `dirty` est levé.

Les patches du fork vivent tous côté boucle A : navigation directionnelle des
splits (`layout::pick_directional`), rendu de la barre d'onglets
(`renderer/island`), et resync du flag/layout quand la barre apparaît
(`screen`, `application`). Voir `CLAUDE.md` § « Splits — focus directionnel »
et l'historique des commits `feat(...)`/`fix(tabs...)`.
