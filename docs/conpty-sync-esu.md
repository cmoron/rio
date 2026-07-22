# Latence de frappe des TUI sous Windows — ConPTY × synchronized output

Post-mortem du fix `3629954ac6` (`fix/sync-esu-inline-timeout`, publié dans
`mnc-v0.4.10-1`). Symptôme d'origine : dans Codex sous Rio Windows, les
caractères semblaient s'afficher à environ 10 FPS alors que le débit, les
animations et le scroll restaient normaux.

## Résumé en trente secondes

Codex redessine son interface sous forme de *frames*. Pour éviter que le
terminal n'affiche une frame à moitié dessinée, il l'entoure de deux séquences
de contrôle :

```text
ESC[?2026h  contenu de la frame  ESC[?2026l
└─ début ─┘                       └── fin ──┘
```

Sous Windows, ConPTY a été observé en train de réémettre d'abord les deux
marqueurs collés, puis le contenu de la frame. Rio recevait donc :

```text
lecture 1 : ESC[?2026h ESC[?2026l
lecture 2 : contenu de la frame
```

Le parseur de Rio armait son délai de sécurité sur le premier marqueur, mais
ne le désarmait pas lorsque le second marqueur était parsé dans la même
lecture. Il prenait alors le contenu suivant pour une frame encore ouverte et
le retenait jusqu'à l'expiration du délai de 150 ms. Le correctif rend le
traitement symétrique : `h` arme, `l` désarme.

## 1. Les acteurs : terminal, TUI, Ratatui, Crossterm et ConPTY

Une TUI (*Terminal User Interface*) ne dessine pas elle-même des pixels. Elle
écrit du texte et des séquences de contrôle dans sa sortie standard. Le
terminal interprète ce flux, met à jour une grille de cellules, puis dessine
cette grille dans une fenêtre.

Dans le cas étudié :

| Élément | Rôle |
|---|---|
| **Codex** | L'application interactive : elle reçoit la touche et décide quoi afficher. |
| **Ratatui** | La bibliothèque de widgets et de rendu : elle construit la prochaine frame en mémoire, la compare à la précédente et ne produit que les cellules modifiées. |
| **Crossterm** | La couche d'I/O terminal utilisée par Codex : lecture clavier et encodage des commandes VT, dont le mode 2026. |
| **ConPTY / ConHost** | Le pseudo-terminal Windows placé entre l'application console et Rio. Il interprète un flux VT dans son propre buffer puis régénère un flux VT vers Rio ; ce n'est pas un simple pipe transparent. |
| **Rio** | L'émulateur de terminal : il lit le flux VT, met à jour sa grille puis la rend avec Sugarloaf. |

Le trajet complet d'une frappe est donc un aller-retour :

```text
entrée
clavier → Rio → ConPTY → Codex

sortie de la nouvelle frame
Codex → Ratatui → Crossterm → ConPTY → parseur Rio → grille → rendu GPU
```

Sous Unix, le PTY transporte principalement un flux d'octets entre les deux
programmes. Sous Windows, la documentation Microsoft décrit bien l'étape
supplémentaire : ConHost applique les sorties de l'application à son propre
buffer, puis son *VT Renderer* génère le texte et les séquences VT envoyés au
terminal. Cela explique pourquoi les limites des `write()` de l'application,
et même l'ordre apparent de certaines opérations de rendu, ne doivent pas être
considérés comme préservés par ConPTY.

## 2. Que signifie exactement `ESC[?2026h` ?

Le nom courant est **synchronized output** ou **synchronized updates**. C'est
une extension de fait utilisant la syntaxe des modes privés DEC ; ce n'est pas
un antique « mode du VT2026 » et `2026` n'est pas une durée.

Décomposition octet par octet :

| Fragment | Sens |
|---|---|
| `ESC` | L'octet `0x1b`, qui commence une séquence d'échappement. En Rust : `\x1b`. |
| `[` | Avec `ESC`, forme le CSI (*Control Sequence Introducer*). |
| `?` | Indique l'espace des modes privés DEC. |
| `2026` | Numéro attribué au mode « synchronized output ». |
| `h` | Active le mode : DECSET, ou **BSU** (*Begin Synchronized Update*) dans le code de Rio. |
| `l` | Désactive le mode : DECRST, ou **ESU** (*End Synchronized Update*). C'est un `l` minuscule, pas le chiffre `1`. |

Chaque marqueur fait huit octets :

```text
1b 5b 3f 32 30 32 36 68  # ESC[?2026h
1b 5b 3f 32 30 32 36 6c  # ESC[?2026l
```

### Le problème que le mode résout

Une frame de TUI contient souvent plusieurs opérations : déplacer le curseur,
effacer une zone, écrire une bordure, changer les couleurs, écrire le texte,
replacer le curseur. Sans synchronisation, le terminal peut rafraîchir sa
fenêtre au milieu de cette série et montrer brièvement une interface déchirée.

Avec le mode 2026, le contrat logique est :

1. `?2026h` : continuer à montrer l'ancienne frame ;
2. appliquer toutes les opérations de la nouvelle frame sans les présenter
   partiellement ;
3. `?2026l` : présenter le nouvel état d'un coup.

Ce mode ne demande donc pas au terminal d'« attendre 150 ms ». Au contraire,
la fin doit rendre la frame disponible immédiatement. Les 150 ms sont
uniquement le filet de sécurité choisi par Rio si une application plante ou
oublie d'envoyer la fin. Rio limite aussi une mise à jour synchronisée à 2 Mio
pour ne pas retenir un flux sans borne.

## 3. Codex utilise-t-il Ratatui, et qui émet réellement le mode 2026 ?

Oui, on peut le savoir avant même d'instrumenter Rio : Codex est open source.
Pour la version installée pendant cette analyse (`codex-cli 0.145.0`) :

1. le manifeste de `codex-tui` dépend explicitement de
   [Ratatui et Crossterm](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/tui/Cargo.toml#L70-L86) ;
2. le rendu de Codex appelle explicitement
   [`stdout().sync_update(...)`](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/tui/src/tui.rs#L895-L951)
   autour de `terminal.draw(...)` ;
3. l'implémentation de `sync_update` dans le fork de Crossterm utilisé par
   Codex écrit un `BeginSynchronizedUpdate`, exécute le rendu, puis écrit un
   `EndSynchronizedUpdate` et vide la sortie
   ([source](https://github.com/nornagon/crossterm/blob/87db8bfa6dc99427fd3b071681b07fc31c6ce995/src/command.rs#L186-L250)) ;
4. ces deux commandes Crossterm sont précisément encodées en
   [`?2026h`](https://github.com/nornagon/crossterm/blob/87db8bfa6dc99427fd3b071681b07fc31c6ce995/src/terminal.rs#L433-L443)
   et
   [`?2026l`](https://github.com/nornagon/crossterm/blob/87db8bfa6dc99427fd3b071681b07fc31c6ce995/src/terminal.rs#L486-L496).

La formulation « c'est ce que fait Ratatui » était toutefois trop large.
Ratatui calcule la frame et son diff
([`previous_buffer.diff(current_buffer)`](https://github.com/nornagon/ratatui/blob/9b2ad1298408c45918ee9f8241a6f95498cdbed2/src/terminal/terminal.rs#L196-L205)),
mais **Codex** choisit ici d'encadrer ce rendu avec le helper de **Crossterm**.
Une application Ratatui qui n'appelle pas ce helper n'émet pas nécessairement
le mode 2026. Le bug de Rio touche toute application qui émet ces séquences,
qu'elle utilise Ratatui ou non.

## 4. Le fonctionnement normal dans Rio

Le cœur se trouve dans `rio-backend/src/performer/handler.rs`. Rio utilise le
même champ, `sync_state.timeout`, comme échéance et comme indicateur « une mise
à jour synchronisée est en cours ».

Dans le cas simple où les lectures du PTY sont séparées :

```text
lecture A : ESC[?2026h
lecture B : FRAME
lecture C : ESC[?2026l
```

le déroulé attendu est :

1. la lecture A passe dans le parseur VT normal ; le dispatch CSI `h` arme
   l'échéance à maintenant + 150 ms ;
2. puisque l'échéance est armée, la lecture B passe dans `advance_sync` et est
   ajoutée à `sync_state.buffer` au lieu d'être appliquée à la grille ;
3. la lecture C est ajoutée au même buffer ; `advance_sync_csi` y reconnaît
   l'ESU exact ;
4. `stop_sync_internal` rejoue le buffer dans le parseur, met à jour la grille,
   efface le buffer et désarme l'échéance ;
5. le thread PTY signale les dégâts au renderer, qui peut présenter la frame.

Les frontières de lecture ne sont cependant pas des frontières de messages.
Le système peut aussi fournir toute la frame dans une seule lecture :

```text
lecture A : ESC[?2026h FRAME ESC[?2026l
```

Dans ce cas, `Processor::advance` choisit le chemin « parseur normal » au début
de l'appel, lorsque l'échéance est encore désarmée. Le même appel parse donc le
BSU, la frame et l'ESU **inline**, sans repasser par `advance_sync`. C'est ce
chemin inline que l'ancien code ne terminait pas correctement.

## 5. La panne pas à pas avec ConPTY

Le relais ConPTY instrumenté a observé la transformation suivante :

```text
écriture de l'application : BSU + FRAME + ESU
sortie de ConPTY, chunk 1 : BSU + ESU
sortie de ConPTY, chunk 2 : FRAME
```

Avant le correctif, le premier chunk suivait ce chemin :

| Étape | `timeout` | Conséquence |
|---|---:|---|
| avant le BSU | désarmé | Rio choisit le parseur normal pour tout le chunk |
| après `?2026h` | armé | comportement attendu |
| après `?2026l` inline | **encore armé** | bug : le dispatch `l` ne nettoie rien |
| arrivée de `FRAME` | armé | Rio la place dans `sync_state.buffer` |
| 150 ms sans autre sortie | expire | `stop_sync` rejoue enfin la frame et elle devient visible |

Une frappe isolée déclenche une seule nouvelle frame, puis l'application attend
la touche suivante : rien ne vient fermer plus tôt l'état fantôme, donc le
délai complet est visible. Avec une animation ou un gros débit, le BSU/ESU de
la frame suivante fait rejouer le buffer précédent avant l'expiration ; le bug
est alors largement masqué.

ConPTY a déjà détruit l'atomicité de la frame lorsqu'il place ses marqueurs
avant son contenu. Le correctif de Rio ne peut pas reconstruire une association
qui n'existe plus dans le flux reçu. Il restaure la justesse de l'état du
parseur et la faible latence ; il ne prétend pas rendre de nouveau atomique la
frame réordonnée par ConPTY.

## 6. Comment la piste 2026 / Ratatui a été trouvée

La piste ne vient pas d'une supposition « les TUI Ratatui sont lentes ». Elle
vient d'une élimination couche par couche :

1. **Le clavier n'était pas lent.** Une touche injectée arrivait à
   l'application en 1 à 2 ms.
2. **Codex répondait vite.** Le relais PTY voyait sa sortie environ 20 ms après
   la touche.
3. **Le rendu général n'était pas saturé.** Le débit restait d'environ 15 MB/s,
   le pacing à 30 FPS était exact et l'adaptateur GPU correct.
4. **`cat` était immédiat dans la même fenêtre, Codex non.** La différence
   pertinente était donc le contenu du flux produit par l'application, pas la
   fenêtre, le clavier ou le GPU.
5. **Le code public de Codex donnait le protocole exact.** Sa TUI utilise
   Ratatui, mais surtout son `Tui::draw` appelle le `sync_update` de Crossterm,
   qui produit `?2026h/l`.
6. **Le relais horodaté puis l'hôte ConPTY minimal ont confirmé les octets.**
   Ils ont montré la paire collée avant le contenu de la frame.

Sans lire le code source de l'application, la même piste se trouve en capturant
sa sortie PTY et en cherchant les octets suivants :

```text
1b 5b 3f 32 30 32 36 68  # début
1b 5b 3f 32 30 32 36 6c  # fin
```

Les mesures historiques qui ont servi au diagnostic étaient :

- Codex réel, injection `PostMessage` + capture `PrintWindow` : environ
  **206 ms** entre la touche et le changement visible ; `cat` : **0 ms** à la
  résolution de la sonde ;
- application minimale : **178 ms** de médiane avec le wrap 2026 contre
  **44 ms** sans le wrap ;
- sonde directe du backend, paire collée puis requête DSR après environ 50 ms :
  **93–99 ms** avant le fix contre environ **2 ms** après.

Le DSR (`ESC[6n`, demande de position du curseur) avait aussi créé un faux
signal pendant les premières mesures Windows : ConPTY peut y répondre dans sa
propre couche console. Une mesure effectuée depuis l'application enfant ne
prouve donc pas nécessairement que la requête a traversé le parseur de Rio. La
sonde rouge/vert utile injectait le motif au niveau du backend, sans dépendre
du rendu GUI ni de la transformation ConPTY.

## 7. Reproduire facilement

### Reproduction visuelle fidèle sous Windows

Il faut lancer les commandes dans **Rio Windows utilisant ConPTY**, par
exemple dans son shell WSL. Avec un binaire Rio antérieur à `3629954ac6`, les
deux boucles suivantes ne se comportent pas pareil (`Ctrl-C` pour sortir) :

```bash
# Contrôle : écho sans synchronized output, immédiat.
while IFS= read -rsn1 key; do printf '\rSans 2026 : %s' "$key"; done

# Reproduction : un seul printf contient BSU + contenu + ESU.
while IFS= read -rsn1 key; do
  printf '\e[?2026h\rAvec 2026 : %s\e[?2026l' "$key"
done
```

Attendu :

- Rio avant le fix : la première boucle est immédiate, la seconde affiche
  chaque touche avec environ 150 à 200 ms de retard ;
- Rio à partir de `mnc-v0.4.10-1` : les deux boucles sont immédiates ;
- un autre terminal n'exposant pas la combinaison ConPTY + ancien parseur Rio
  peut ne montrer aucun écart.

Cette reproduction est volontairement qualitative. Pour comparer deux builds,
utiliser le parent `3629954ac6^` comme version rouge et `3629954ac6` comme
version verte évite de mélanger le fix avec d'autres changements de `mnc`.

### Reproduction déterministe du bug de parseur

Le premier test ajouté par le correctif encode exactement la sortie observée
de ConPTY, sans Windows ni chronométrage :

```text
advance(BSU + ESU)  → l'échéance doit être désarmée
advance("abc")      → abc doit être parsé tout de suite, buffer vide
```

Les trois scénarios se lancent séparément ainsi :

```bash
cargo test -p rio-backend sync_esu_inline_clears_pending_timeout
cargo test -p rio-backend sync_esu_followed_by_new_bsu_stays_pending
cargo test -p rio-backend sync_split_esu_still_flushes
```

Ils couvrent respectivement la paire inline de ConPTY, un nouvel update qui
commence juste après le précédent, et le chemin historique où l'ESU arrive
dans une lecture séparée.

## 8. La cause et le correctif dans le diff

Le commit contient 109 lignes ajoutées, mais seulement deux changements de
logique ; le reste est constitué de commentaires et de trois tests.

1. Dans le dispatch CSI `('l', [b'?'])`, si le paramètre vaut 2026,
   `clear_timeout()` désarme l'état. C'est le miroir nécessaire du dispatch
   `('h', [b'?'])`, qui l'arme.
2. Dans `stop_sync_internal`, branche `Some(bsu_offset)`, le code réarme
   explicitement le délai. Cette branche signifie que le buffer rejoué
   contenait la fin de l'update courant, mais qu'un **nouveau** BSU reste dans
   la partie conservée. Le replay peut maintenant rencontrer un ESU inline et
   désarmer le délai grâce au premier changement ; sans le second, le nouvel
   update serait laissé ouvert dans le buffer mais marqué comme inactif.

### Revue du correctif

Aucun défaut bloquant n'a été trouvé dans le diff :

- le fix est placé au point commun, dans le dispatch du mode 2026, et non dans
  un chemin Windows ou un appelant particulier ;
- le second changement protège correctement l'état imbriqué produit pendant
  le replay ;
- les tests couvrent le bug, l'interaction entre les deux changements et le
  chemin antérieur qui devait rester valide ;
- le chemin de production n'ajoute ni `unsafe`, ni allocation, ni dépendance.

Deux limites sont intentionnelles et préexistantes : pendant une mise à jour
déjà bufferisée, `advance_sync_csi` ne reconnaît que les formes exactes
`ESC[?2026h` et `ESC[?2026l`, pas une liste combinant plusieurs modes ; et les
tests unitaires valident le parseur, pas la transformation propre à une version
de Windows/ConPTY. La boucle WSL ci-dessus complète ce dernier point.

## 9. Impact hors Windows

La réorganisation spectaculaire est propre au chemin ConPTY observé, mais le
défaut d'état était multiplateforme. Sur un PTY Unix, une lecture peut tout à
fait contenir `BSU + FRAME + ESU` en une fois. L'ensemble est alors parsé inline
et l'ancien dispatch `l` laissait également l'échéance armée.

Une sortie non wrappée arrivant juste après pouvait donc être retenue jusqu'à
150 ms. Avec un flux continu, une séquence 2026 suivante pouvait vider plus tôt
le buffer précédent, ce qui rendait l'effet discret. À défaut d'autre sortie,
le thread PTY effectuait un réveil inutile à l'expiration pour chaque frame
inline ayant laissé cet état fantôme. Le fix est donc un correctif de justesse
multiplateforme, avec un gain visible surtout sous Windows.

## Références

- [Spécification historique des synchronized updates](https://gitlab.com/gnachman/iterm2/-/wikis/synchronized-updates-spec)
- [Microsoft Learn — `CreatePseudoConsole`](https://learn.microsoft.com/en-us/windows/console/createpseudoconsole)
- [Microsoft — architecture et traduction VT de ConPTY](https://devblogs.microsoft.com/commandline/windows-command-line-introducing-the-windows-pseudo-console-conpty/)
- [Codex 0.145.0 — dépendances de la TUI](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/tui/Cargo.toml#L70-L86)
- [Codex 0.145.0 — synchronized update autour du rendu](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/tui/src/tui.rs#L895-L951)
