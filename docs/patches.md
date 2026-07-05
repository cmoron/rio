# Patches du fork — inventaire & stratégie upstream

> Statut upstream vérifié le **2026-07-05** (`gh` sur `raphamorim/rio`). Les
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
| ⚫ **Jamais upstream** | Tuyauterie de fork (CI, doc mnc) |

## Vue d'ensemble

| Commit | Patch | Décision | Statut upstream |
|---|---|---|---|
| `eb89880` | feat(splits): focus directionnel h/j/k/l | 🟢 Perso | Aucune PR/issue upstream |
| `7005be6` | fix(macos): ⌘H n'est plus mangé par le menu Hide | 🟢 Perso | Aucune PR/issue upstream |
| `7f89076` | feat(navigation): display-tab-number | 🔵 Proposé | **PR #1697 ouverte** (nous) |
| `ee589ea` | fix(tabs): recompute grid rows au toggle barre | 🟠 Porté | PR #1699 **fermée** (doublon) → #1632 / #1687 ouvertes |
| `8a03a01` | fix(layout): refresh Taffy root si marge change | 🟠 Porté | Adopte **PR #1632** (@nikicat, ouverte) |
| `53b14fd` | fix(tabs): réserver la bande island à 3→2 onglets | 🟠 Porté | Bug latent upstream aussi ; pas de PR |
| `a64dc54` | fix(tabs): snap bordures aux pixels physiques | 🟠 Porté | **PR #1682** (@cantona, ouverte) |
| `5d7e890` | fix(tabs): repaint après fermeture via shell exit | 🟠 Porté | **PR #1585** (@ddidderr, ouverte) |
| `5644e82` `b4ddd51` `01094168` `c1a84b2` `5fb677d` | ci(mnc/perso): build/release macOS+Windows | ⚫ Jamais | — |
| `804c07` (+ ces docs) | docs(mnc): workflow, archi, patches | ⚫ Jamais | — |

---

## 🟢 Perso — features sans équivalent upstream

### `eb89880` — Focus directionnel entre splits (h/j/k/l)
Actions `selectsplit{left,right,up,down}` : focus **spatial** entre panneaux
(voisin le plus proche avec recouvrement d'axe), là où `selectnext/prevsplit`
cyclique ne suffit pas en grille 2D. Bindables au clavier, **sans keybinding
par défaut** (évite les conflits). Cœur : `layout::pick_directional`.
- **Statut :** aucune PR upstream (`directional`, `selectsplit`, `vim split`,
  `focus pane` → rien de correspondant). Candidat propre à upstreamer si envie.

### `7005be6` — macOS : ⌘H atteint les bindings config
macOS résout les key-equivalents du menu principal **avant** que `keyDown`
n'atteigne le moteur de bindings de Rio : un `[bindings]` sur ⌘H (ex.
`selectsplitleft`) ne se déclenchait jamais, le menu lançait Hide. On retire
l'équivalent ⌘H de l'item Hide (toujours cliquable) → ⌘H atteint la config.
- **Touche `rio-window`** (le fork de winit), `platform_impl/macos/menu.rs`.
- **Statut :** aucune PR/issue upstream trouvée. Perso ; upstreamable.

---

## 🔵 Proposé — PR ouverte à notre nom

### `7f89076` — display-tab-number
Préfixe optionnel `1 vim`, `2 htop`… (façon ghostty). Le numéro dérive de la
**position de rendu** → réordonner les onglets renumérote gratuitement. Config
`display-tab-number` + `tab-number-separator`. Mode `Tab` uniquement (pas
`NativeTab`).
- **Statut :** **[PR #1697](https://github.com/raphamorim/rio/pull/1697)
  ouverte**, en attente de review (aucun commentaire au 2026-07-05).

---

## 🟠 Porté en attendant — un upstream existe déjà (non mergé)

Ces patches corrigent de vrais bugs, mais un équivalent upstream **non mergé**
existe. On garde notre version dans `mnc` pour l'usage quotidien ; **on droppe
dès que l'upstream fusionne** (réconciliation au prochain rebase sur `main`).

### Saga « marge de la barre d'onglets » — issue #1495
Ouvrir/fermer un onglet montre/cache la barre → la marge top change → les
lignes PTY n'étaient pas recalculées → le bas débordait hors fenêtre jusqu'au
prochain resize. Trois commits couvrent les trois chemins :

- **`ee589ea`** — chemin `resize_top_or_bottom_line` (toggle barre d'onglets /
  barre de recherche). Notre **PR #1699 a été fermée par nous-mêmes comme
  doublon** de #1495, déjà adressée par #1632 et #1687 (toutes deux ouvertes).
- **`8a03a01`** — chemin hot-reload de config. **Adopte la PR #1632**
  (@nikicat) : `update_scaled_margin` laissait le root Taffy sur l'ancienne
  aire. « À dropper quand #1632 est mergé. »
- **`53b14fd`** — `close_tab` lisait `ctx().len()` *après* retrait du contexte
  puis re-soustrayait 1 → fermer de 3 à 2 onglets passait `num_tabs=1` et
  retirait à tort la bande `ISLAND_HEIGHT`. Bug latent upstream aussi, réveillé
  seulement une fois le recompute des lignes actif. Inerte sur macOS.

> Détail mémoire : la marge vit à **deux endroits** — `scaled_margin` (offset
> de rendu) et le root Taffy (layout). Seul `resize()` resynchronise les deux
> *et* recompte les lignes. Cf. mémoire `rio-margin-taffy-desync`.

### `a64dc54` — Snap des bordures d'onglets aux pixels physiques
Dessin en coords logiques → hairline à cheval sur 2 pixels physiques (bordures
floues/qui disparaissent). Fix : arrondi à la grille physique via le scale
factor (`snap_to_physical_pixel`, `physical_pixel_span`).
- **Statut :** **[PR #1682](https://github.com/raphamorim/rio/pull/1682)**
  (@cantona) ouverte, même sujet. Porté en attendant.

### `5d7e890` — Repaint après fermeture d'onglet via shell exit
Contexte retiré mais rien ne marquait `dirty` ; `render()` est gated dessus →
l'onglet fermé persistait. Fix : `request_overlay_redraw()` dans
`application.rs`.
- **Statut :** **[PR #1585](https://github.com/raphamorim/rio/pull/1585)**
  (@ddidderr) ouverte, même sujet. Porté en attendant.

---

## ⚫ Jamais upstream

### `ci(mnc)` / `ci(perso)` — pipeline de build/release du fork
`5644e82` `b4ddd51` `01094168` `c1a84b2` `5fb677d` : build/release macOS arm64
+ Windows MSI (sans GoReleaser), préfixe `ci(mnc)`. Pure tuyauterie de fork,
**jamais** proposée upstream.

### `docs(mnc)` — documentation interne
`804c07` (workflow, deps, map) + `docs/architecture.md` + ce fichier.

---

## Rappel workflow

- Un patch destiné upstream = **branche depuis `main`** → PR (base propre).
- Intégration locale dans `mnc` par **merge** (octopus si plusieurs), jamais
  cherry-pick.
- Ne **jamais** proposer les commits `ci(mnc)` upstream.
- Un patch 🟠 se supprime de `mnc` quand son équivalent upstream merge : ne pas
  le maintenir en double éternellement.
