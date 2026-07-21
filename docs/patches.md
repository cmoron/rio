# Patches du fork — inventaire & stratégie upstream

> Statut upstream vérifié le **2026-07-19** (`gh` sur `raphamorim/rio`). Les
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
| `3629954` | fix(performer): sync-update timeout collé (ConPTY × 2026) | 🟢 Perso | Aucune PR/issue upstream — **prêt à proposer** |
| `24f6685` | feat(splits): focus directionnel h/j/k/l | 🟢 Perso | Aucune PR/issue upstream |
| `feed9f5` | fix(macos): ⌘H n'est plus mangé par le menu Hide | 🟢 Perso | Aucune PR/issue upstream |
| `1089a1a` | fix(tabs): initialiser un onglet depuis la taille fenêtre | 🟢 Perso | Correctif absent de `upstream/main` |
| `5793f97` | feat(navigation): display-tab-number | 🔵 Proposé | **PR #1697 ouverte et mergeable** (head `5be6540`) |
| `f958a02` | fix(tabs): recompute grid rows au toggle barre | 🟠 Porté | PR #1699 **fermée** (doublon) → #1632 / #1687 ouvertes |
| `8e2f8ec` | fix(layout): refresh Taffy root si marge change | 🟠 Porté | Adopte **PR #1632** (@nikicat, ouverte) |
| `b7f57c6` | fix(tabs): repaint après fermeture via shell exit | 🟠 Porté | **PR #1585** (@ddidderr, ouverte) |
| ancien `53b14fd` | fix(tabs): réserver la bande island à 3→2 onglets | ✅ Retiré | Intégré upstream dans `c8bbf459` |
| ancien `a64dc54` | fix(tabs): snap bordures aux pixels physiques | ✅ Retiré | Bordures supprimées upstream dans `c8bbf459` |
| `653eb04` `dba5146` `d289fa6` `a70453c` `b018952` | ci(mnc/perso): build/release macOS+Windows | ⚫ Jamais | — |
| `a3d5f26` (+ ces docs) | docs(mnc): workflow, archi, patches | ⚫ Jamais | — |

---

## 🟢 Perso — features sans équivalent upstream

### `3629954` — Latence de frappe TUI sous Windows (ConPTY × 2026)
ConPTY ré-émet la paire `?2026h?2026l` collée avant le contenu de la frame ;
l'ESU parsé inline ne désarmait pas `sync_state.timeout` → chaque frappe
bufferisée jusqu'au timeout de 150 ms. Fix : clear dans le dispatch `l` +
ré-armement dans `stop_sync_internal` (nouveau BSU). Bug latent sur toutes
les plateformes, symptôme majeur sur Windows. Détail complet (repro, sondes,
mesures 93 ms→2 ms) : [`conpty-sync-esu.md`](conpty-sync-esu.md).
- **Statut :** aucune PR/issue upstream (vérifié 2026-07-21). Branche
  `fix/sync-esu-inline-timeout` prête ; PR à ouvrir après accord.

### `24f6685` — Focus directionnel entre splits (h/j/k/l)
Actions `selectsplit{left,right,up,down}` : focus **spatial** entre panneaux
(voisin le plus proche avec recouvrement d'axe), là où `selectnext/prevsplit`
cyclique ne suffit pas en grille 2D. Bindables au clavier, **sans keybinding
par défaut** (évite les conflits). Cœur : `layout::pick_directional`.
- **Statut :** aucune PR upstream (`directional`, `selectsplit`, `vim split`,
  `focus pane` → rien de correspondant). Candidat propre à upstreamer si envie.

### `feed9f5` — macOS : ⌘H atteint les bindings config
macOS résout les key-equivalents du menu principal **avant** que `keyDown`
n'atteigne le moteur de bindings de Rio : un `[bindings]` sur ⌘H (ex.
`selectsplitleft`) ne se déclenchait jamais, le menu lançait Hide. On retire
l'équivalent ⌘H de l'item Hide (toujours cliquable) → ⌘H atteint la config.
- **Touche `rio-window`** (le fork de winit), `platform_impl/macos/menu.rs`.
- **Statut :** aucune PR/issue upstream trouvée. Perso ; upstreamable.

### `1089a1a` — Nouvel onglet initialisé depuis la taille fenêtre
Après un layout Taffy, la dimension du panneau exclut les marges. La réutiliser
pour créer l'onglet suivant retire une marge supplémentaire à chaque génération.
Le nouveau contexte part donc de `grid_dimension()`, qui conserve la taille de
la fenêtre.
- **Statut :** `upstream/main` réutilise encore `current.dimension` hors splits ;
  patch conservé localement.

---

## 🔵 Proposé — PR ouverte à notre nom

### `5793f97` — display-tab-number
Préfixe optionnel `1 vim`, `2 htop`… (façon ghostty). Le numéro dérive de la
**position de rendu** → réordonner les onglets renumérote gratuitement. Config
`display-tab-number` + `tab-number-separator`. Mode `Tab` uniquement (pas
`NativeTab`).
- **Statut :** **[PR #1697](https://github.com/raphamorim/rio/pull/1697)
  ouverte et mergeable**, sans review ni commentaire. Branche rebasée sur
  `upstream/main` le 2026-07-19 (`5be6540`) après le refactor tabs `c8bbf459` ;
  CI upstream relancée.

---

## 🟠 Porté en attendant — un upstream existe déjà (non mergé)

Ces patches corrigent de vrais bugs, mais un équivalent upstream **non mergé**
existe. On garde notre version dans `mnc` pour l'usage quotidien ; **on droppe
dès que l'upstream fusionne** (réconciliation au prochain rebase sur `main`).

### Saga « marge de la barre d'onglets » — issue #1495
Ouvrir/fermer un onglet montre/cache la barre → la marge top change → les
lignes PTY n'étaient pas recalculées → le bas débordait hors fenêtre jusqu'au
prochain resize. Deux commits locaux restent nécessaires :

- **`f958a02`** — chemin `resize_top_or_bottom_line` (toggle barre d'onglets /
  barre de recherche). Notre **PR #1699 a été fermée par nous-mêmes comme
  doublon** de #1495, déjà adressée par #1632 et #1687 (toutes deux ouvertes).
- **`8e2f8ec`** — chemin hot-reload de config. **Adopte la PR #1632**
  (@nikicat) : `update_scaled_margin` laissait le root Taffy sur l'ancienne
  aire. « À dropper quand #1632 est mergé. »

Le chemin 3→2 onglets de l'ancien `53b14fd` est désormais couvert upstream :
`close_tab` passe directement le nombre restant depuis `c8bbf459`.

> Détail mémoire : la marge vit à **deux endroits** — `scaled_margin` (offset
> de rendu) et le root Taffy (layout). Seul `resize()` resynchronise les deux
> *et* recompte les lignes. Cf. mémoire `rio-margin-taffy-desync`.

### `b7f57c6` — Repaint après fermeture d'onglet via shell exit
Contexte retiré mais rien ne marquait `dirty` ; `render()` est gated dessus →
l'onglet fermé persistait. Fix : `request_overlay_redraw()` dans
`application.rs`.
- **Statut :** **[PR #1585](https://github.com/raphamorim/rio/pull/1585)**
  (@ddidderr) ouverte, même sujet. Porté en attendant.

---

## ✅ Retirés au rebase du 2026-07-19

- **`53b14fd`** — le refactor tabs upstream `c8bbf459` passe déjà le vrai
  nombre d'onglets restant à `resize_top_or_bottom_line`.
- **`a64dc54`** — le même refactor remplace la barre et ses séparateurs par des
  islands arrondies sans les bordures concernées. La PR tierce #1682 reste
  ouverte mais conflictuelle ; reporter notre pixel-snap n'aurait plus de cible.

---

## ⚫ Jamais upstream

### `ci(mnc)` / `ci(perso)` — pipeline de build/release du fork
`653eb04` `dba5146` `d289fa6` `a70453c` `b018952` : build/release macOS arm64
+ Windows MSI (sans GoReleaser), préfixe `ci(mnc)`. Pure tuyauterie de fork,
**jamais** proposée upstream.

### `docs(mnc)` — documentation interne
`a3d5f26` (workflow, deps, map) + `docs/architecture.md` + ce fichier.

---

## Rappel workflow

- Un patch destiné upstream = **branche depuis `main`** → PR (base propre).
- Intégration locale dans `mnc` par **merge** (octopus si plusieurs), jamais
  cherry-pick.
- Ne **jamais** proposer les commits `ci(mnc)` upstream.
- Un patch 🟠 se supprime de `mnc` quand son équivalent upstream merge : ne pas
  le maintenir en double éternellement.
