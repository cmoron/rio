# Rio — fork perso

Fork de [rio](https://github.com/raphamorim/rio) (terminal en Rust).

## Workflow git (fork)

- **Branche de feature depuis `main`** (`origin/main`, miroir upstream) pour
  tout changement de code destiné à partir upstream.
- Commit propre et atomique sur cette branche → sert de **PR upstream**.
- Pour l'**usage local**, intégrer les branches de feature dans **`mnc`** par
  **merge** (`git merge --no-ff`, **octopus** quand plusieurs d'un coup) — jamais
  de cherry-pick. `mnc` est une branche d'intégration jetable, reconstructible à
  partir de `main` + les branches perso.
- `mnc` = `main` + commits/branches perso : CI build/release (macOS arm64 +
  Windows MSI, préfixe `ci(mnc):`) + fixes locaux. **Ne jamais proposer les
  commits `ci(mnc)` upstream.**
- Règles git globales (hors `mnc`) : rebase, Conventional Commits, pas de
  `--no-verify`, pas de force push hors branche perso. L'intégration dans `mnc`
  est la seule exception : elle se fait par merge (octopus), pas par rebase.

## Plateformes cibles & config

- **Cibles buildées : macOS (arm64) et Windows (MSI).** Linux non buildé pour
  l'instant (la WSL sert uniquement de validation locale, cf. Build).
- Sur Windows, Rio tourne **nativement (hors WSL)** et sert à *invoquer* une
  session WSL. La config vit donc côté Windows, **pas** dans le FS de la WSL.
- Chemins de `config.toml` résolus par `config_dir_path()`
  (`rio-backend/src/config/mod.rs`) :
  - macOS : `~/.config/rio/` (⚠️ codé en dur, **ignore** `XDG_CONFIG_HOME`)
  - Windows : `%USERPROFILE%\AppData\Local\rio\`
  - Linux/autres : `$XDG_CONFIG_HOME/rio` sinon `~/.config/rio`
  - Override universel : variable `$RIO_CONFIG_HOME`.
- Sous-dossiers : `themes/`, `log/`. Config rechargée à chaud (`watcher.rs`).

## Map du projet (workspace Cargo)

`frontends/rioterm` est **le binaire** ; le reste sont des libs, dont plusieurs
forks de crates upstream maintenus pour Rio.

| Crate | Rôle |
|---|---|
| `frontends/rioterm` | **Le binaire** : terminal GPU — lifecycle, fenêtres, rendu, splits, input. |
| `rio-backend` | Cœur non-UI : parsing ANSI, modèle de grille, config, events, graphics, clipboard, sélection. |
| `sugarloaf` | Moteur de rendu (WebGPU ; desktop + WASM). |
| `teletypewriter` | Création de pty/tty (spawn du shell). |
| `rio-window` | Fork de `winit` (fenêtrage). |
| `corcovado` | IO non-bloquante (fork `mio`). |
| `rio-grapheme-width` | Tables largeur emoji/grapheme (fork `wezterm-char-props`). |
| `rio-notifier` | Notifications. |
| `frontends/wasm` | Cible web (hors périmètre perso). |

Les deux crates qu'on touche le plus :

- **`frontends/rioterm/src/`** — `application.rs` (lifecycle), `router/`
  (fenêtres), `screen/` (écran terminal), `renderer/` (command palette, search,
  scrollbar, curseurs…), `layout/` (splits/panneaux, cf. `pick_directional`),
  `context/` (contexte + titre), `bindings/` (keybindings + kitty keyboard),
  `mouse/`, `ime.rs`, `hints.rs`, `cli.rs`, `watcher.rs` (hot-reload config).
- **`rio-backend/src/`** — `config/` (parsing `config.toml`, chemins, thèmes,
  bindings), `ansi/` (parseur d'échappement), `crosswords/` (buffer/grille = le
  modèle d'état du terminal), `performer/` (applique les séquences à la grille),
  `graphics/` (sixel/kitty), `event/`, `selection.rs`, `clipboard.rs`.

## Dev — commandes

    make lint    # cargo fmt --check + clippy --all-targets --all-features -D warnings
    make test    # lint puis cargo test --release

## Build (Linux / WSL)

Deps système (union des CI test + release) :

    sudo apt-get install -y libasound2-dev libfontconfig1-dev glslang-tools libwayland-dev pkg-config

Sans elles : `sugarloaf` échoue (pas de compilateur GLSL→SPIR-V) et
`yeslogic-fontconfig-sys` échoue (`fontconfig.pc` introuvable). Build habituel
sur macOS + Windows ; la WSL sert de validation locale.

    cargo test -p rioterm    # binaire, pas une lib : pas de --lib

## Splits — focus directionnel (patch fork)

Actions `selectsplit{left,right,up,down}` (config `[bindings]`) : focus spatial
entre panneaux (voisin le plus proche avec recouvrement d'axe), là où
`selectnext/prevsplit` cyclique ne suffit pas en grille 2D. Bindables au
clavier, sans keybinding par défaut (évite les conflits). Cf. `pick_directional`
dans `frontends/rioterm/src/layout/mod.rs`.
