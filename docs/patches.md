# Patches du fork — inventaire & stratégie upstream

> Statut upstream vérifié le **2026-10-20** (`upstream/main` `59203cc`, Rio
> 0.5.28, et `gh` sur `raphamorim/rio`). Les
> états de PR évoluent : re-checker avant de décider d'un push. Source de
> vérité complémentaire = les messages de commit eux-mêmes (le « pourquoi » y
> est souvent inline). Pour l'architecture du code, voir
> [`architecture.md`](architecture.md).

`mnc` = `main` + ces commits. Branche d'intégration **jetable**, reconstruite
à partir de `main` + les branches perso (cf. [`../CLAUDE.md`](../CLAUDE.md)).

## Taxonomie des décisions

| Catégorie | Sens |
|---|---|
| 🟢 **Perso** | Feature de préférence, aucun équivalent upstream — on la garde, à proposer un jour ou non |
| 🔵 **Proposé** | PR ouverte upstream à notre nom, en attente de review |
| 🟠 **Porté en attendant** | Un équivalent upstream existe déjà (PR d'un tiers **non mergée**, ou la nôtre fermée comme doublon) — on garde notre version localement, **à dropper quand l'upstream merge**. Pas re-proposé. |
| ✅ **Retiré** | Correctif désormais présent upstream ou rendu sans objet par son refactor — absent de `mnc` |
| ⚫ **Jamais upstream** | Tuyauterie de fork (CI, doc mnc) |

## Vue d'ensemble

| Commit | Patch | Décision | Statut upstream |
|---|---|---|---|
| `f52732e` | feat(splits): focus directionnel h/j/k/l | 🟢 Perso | Aucune PR ; demande ouverte **#1549** |
| `81bcce7` | fix(macos): ⌘H n'est plus mangé par le menu Hide | 🟢 Perso | Aucune PR/issue upstream |
| `c5bcb01` | fix(tabs): initialiser un onglet depuis la taille fenêtre | 🟢 Perso | Correctif absent de `upstream/main` |
| `b534db8` | fix(sugarloaf): retenter la frame si l'acquire wgpu échoue | 🟢 Perso | Aucune PR ; upstreamable |
| `2d287b6` `b3933e6` | fix(teletypewriter): réveils Condvar perdus (pipes Windows) | 🟢 Perso | Aucune PR ; upstreamable |
| `9e82687` | feat(navigation): display-tab-number | 🔵 Proposé | **PR #1697 ouverte**, rebasée sur Rio 0.5.28 |
| `99678d8` | fix(tabs): repaint après fermeture via shell exit | 🟠 Porté | **PR #1585** (@ddidderr, ouverte) |
| ancien `a11479e` | fix(hints): offsets Onig → colonnes (hover) | ✅ Retiré | Intégré upstream par **PR #1884** |
| ancien `dc89a82` | fix(performer): sync-update timeout collé (ConPTY × 2026) | ✅ Retiré | Intégré upstream par **PR #1789** ; notre PR #1754 fermée |
| ancien `f958a02` `8e2f8ec` | fix(tabs/layout): marge et grille au toggle barre | ✅ Retiré | Intégré upstream par **PR #1723** |
| ancien `53b14fd` `a64dc54` | fix(tabs): island 3→2 onglets, snap bordures | ✅ Retiré | Absorbé par le refactor upstream `c8bbf459` |
| `730b9ed` | ci(mnc): build/release macOS + Windows MSI | ⚫ Jamais | — |
| `84c4b45` | chore(mnc): AGENTS + outillage du fork | ⚫ Jamais | — |
| `141f158` | chore(mnc): Clippy strict propre sous Linux | ⚫ Jamais | Quality gate du fork |
| `db76218` (+ ces docs) | docs(mnc): workflow, archi, patches | ⚫ Jamais | — |

---

## 🟢 Perso — features sans équivalent upstream

### `f52732e` — Focus directionnel entre splits (h/j/k/l)
Actions `selectsplit{left,right,up,down}` : focus **spatial** entre panneaux
(voisin le plus proche avec recouvrement d'axe), là où `selectnext/prevsplit`
cyclique ne suffit pas en grille 2D. Bindables au clavier, **sans keybinding
par défaut** (évite les conflits). Cœur : `layout::pick_directional`.
- **Statut :** aucune PR upstream ; la demande **#1549** (navigation
  directionnelle entre splits) est ouverte. Candidat propre à upstreamer.

### `81bcce7` — macOS : ⌘H atteint les bindings config
macOS résout les key-equivalents du menu principal **avant** que `keyDown`
n'atteigne le moteur de bindings de Rio : un `[bindings]` sur ⌘H (ex.
`selectsplitleft`) ne se déclenchait jamais, le menu lançait Hide. On retire
l'équivalent ⌘H de l'item Hide (toujours cliquable) → ⌘H atteint la config.
- **Touche `rio-window`** (le fork de winit), `platform_impl/macos/menu.rs`.
- **Statut :** aucune PR/issue upstream trouvée. Perso ; upstreamable.

### `c5bcb01` — Nouvel onglet initialisé depuis la taille fenêtre
Après un layout Taffy, la dimension du panneau exclut les marges. La réutiliser
pour créer l'onglet suivant retire une marge supplémentaire à chaque génération.
Le nouveau contexte part donc de `grid_dimension()`, qui conserve la taille de
la fenêtre.
- **Statut :** `upstream/main` réutilise encore `current.dimension` hors splits ;
  patch conservé localement.

### `b534db8` — Retenter la frame quand l'acquire de surface wgpu échoue

`render_wgpu` abandonnait silencieusement la frame quand `get_current_texture`
échouait, après que l'embedder avait déjà consommé les damages : écran figé
jusqu'au prochain trafic PTY (DX12, autour des move/resize). La perte passe
désormais par `frame_dropped`, comme le chemin Metal.
- **Statut :** aucune PR upstream. Upstreamable.

### `2d287b6` `b3933e6` — Réveils Condvar perdus dans les pipes Windows

Les workers `EventedAnonRead`/`EventedAnonWrite` testaient leur prédicat hors
mutex puis attendaient ; une notification tombant entre les deux était perdue
et le worker dormait pour toujours → TUI gelée après move/resize sous Windows.
Test de stress + fix.
- **Statut :** aucune PR/issue upstream. Upstreamable.

---

## 🔵 Proposé — PR ouverte à notre nom

### `9e82687` — display-tab-number
Préfixe optionnel `1 vim`, `2 htop`… (façon ghostty). Le numéro dérive de la
**position de rendu** → réordonner les onglets renumérote gratuitement. Config
`display-tab-number` + `tab-number-separator`. Mode `Tab` uniquement (pas
`NativeTab`).
- **Statut :** **[PR #1697](https://github.com/raphamorim/rio/pull/1697)
  ouverte**. Rebasée sur `main` 0.5.28 le 2026-10-20 : le titre de base vient
  désormais de `displayed_title_for_tab` (chaîne partagée avec la titlebar
  native upstream), le préfixe n'est appliqué que dans la barre d'onglets.

---

## 🟠 Porté en attendant — un upstream existe déjà (non mergé)

Ce patch corrige un vrai bug, mais un équivalent upstream **non mergé** existe.
On garde notre version dans `mnc` pour l'usage quotidien ; **on la droppe dès
que l'upstream fusionne** (réconciliation au prochain rebase sur `main`).

### `99678d8` — Repaint après fermeture d'onglet via shell exit
Contexte retiré mais rien ne marquait `dirty` ; `render()` est gated dessus →
l'onglet fermé persistait. Fix : `request_overlay_redraw()` dans
`application.rs`.
- **Statut :** **[PR #1585](https://github.com/raphamorim/rio/pull/1585)**
  (@ddidderr) ouverte, même sujet. Porté en attendant.

---

## ✅ Retirés lors des rebases

### Reconstruction du 2026-10-20 (Rio 0.5.8 → 0.5.28)

`mnc` reconstruite depuis `main` au lieu d'empiler un nouveau rebase (l'ancien
historique contenait chaque patch en 2-3 exemplaires). Sauvegarde locale :
`backup/2026-10-20/*`.

- **`a11479e`** — la [PR #1884](https://github.com/raphamorim/rio/pull/1884)
  ancre désormais les matches hover sur une table octet → cellule
  (`hints.rs`) ; branche `fix/hint-hover-byte-columns` supprimée.
- Les commits `ci`/`chore`/`docs(mnc)` successifs sont consolidés en un commit
  par catégorie.

### Rebase du 2026-08-04

- **`dc89a82`** — le correctif du timeout ESU inline et son complément de flush
  sont désormais upstream via la [PR #1789](https://github.com/raphamorim/rio/pull/1789)
  (`93bff5a`, `7496617`) dans la nouvelle crate `rio-vt`. Notre
  [PR #1754](https://github.com/raphamorim/rio/pull/1754) a été fermée et la
  branche locale supprimée.
- **`5e11b1d` / `daaa438`** — nettoyages ponctuels de la quality gate désormais
  absorbés par upstream ou sans objet après l'extraction de `rio-vt`.

### Rebase du 2026-07-22

- **`f958a02`** — le chemin `resize_top_or_bottom_line` est désormais corrigé
  upstream par la [PR #1723](https://github.com/raphamorim/rio/pull/1723), qui
  met à jour la marge et les dimensions de toutes les grilles.
- **`8e2f8ec`** — la même PR appelle désormais `try_update_size` depuis
  `update_scaled_margin`, couvrant aussi le hot-reload de configuration.

### Rebase du 2026-07-19

- **`53b14fd`** — le refactor tabs upstream `c8bbf459` passe déjà le vrai
  nombre d'onglets restant à `resize_top_or_bottom_line`.
- **`a64dc54`** — le même refactor remplace la barre et ses séparateurs par des
  islands arrondies sans les bordures concernées. La PR tierce #1682 reste
  ouverte mais conflictuelle ; reporter notre pixel-snap n'aurait plus de cible.

---

## ⚫ Jamais upstream

### `730b9ed` — pipeline de build/release du fork
Build macOS arm64 + release Windows exe/MSI (sans GoReleaser) sur tag
`mnc-v*`. Pure tuyauterie de fork, **jamais** proposée upstream.

### `84c4b45` — outillage du fork

Instructions `AGENTS.md` et cible Makefile `install-macos-arm64`. Conservé
uniquement dans `mnc`.

### `141f158` — quality gate Clippy sous Linux

`make lint` (clippy `-D warnings`) tourne sous Linux/WSL, où upstream laisse
des bindings macOS-only inutilisés et déclenche des lints récents dans le
backend x11. Ajustements mécaniques, sans changement de comportement.

### `docs(mnc)` — documentation interne
`db76218` : `CLAUDE.md`, `docs/architecture.md`, ce fichier.

---

## Rappel workflow

- Un patch destiné upstream = **branche depuis `main`** → PR (base propre).
- Intégration locale dans `mnc` par **merge** (octopus si plusieurs), jamais
  cherry-pick.
- Ne **jamais** proposer les commits `ci(mnc)` upstream.
- Un patch 🟠 se supprime de `mnc` quand son équivalent upstream merge : ne pas
  le maintenir en double éternellement.
