# Latence de frappe des TUI sous Windows — ConPTY × synchronized output

Post-mortem du fix `3629954ac6` (`fix/sync-esu-inline-timeout`, releasé dans
`mnc-v0.4.10-1`). Symptôme d'origine : frappe « à 10 FPS » dans codex (et tout
TUI ratatui) sous Windows **uniquement** — débit, animations et scroll normaux.

## 1. Reproduction

Deux repros, du plus fidèle au plus minimal :

- **Repro minimal GUI** (Windows) : une app de 15 lignes qui, à chaque
  caractère lu, repeint l'écran en wrappant la frame dans
  `ESC[?2026h … ESC[?2026l` (synchronized output, ce que fait ratatui).
  Latence d'écho mesurée : **178 ms** médiane par frappe. La même app sans le
  wrap 2026 : **44 ms**. Le wrap est la seule variable.
- **Sonde CLI, sans GUI, toutes plateformes** : émettre la paire
  `ESC[?2026h ESC[?2026l` collée (voir § 2 : c'est le pattern que ConPTY
  fabrique), attendre ~50 ms, puis envoyer un DSR `ESC[6n` et chronométrer la
  réponse. Rio bugué : **~93-99 ms** (la requête reste bufferisée jusqu'au
  timeout de sync). Rio corrigé : **~2 ms**. C'est le critère rouge/vert du
  fix, indépendant de ConPTY et du rendu.

## 2. Détection — la démarche

Tout ce qui se mesure « en aveugle » était sain, ce qui a longtemps masqué le
bug : débit (~3× plus lent que Windows Terminal mais 15 MB/s, largement
suffisant), rendu paced 30 fps à la frame près, adapter GPU correct (Vulkan
discret), chemin clavier au rythme exact de l'envoi, et même la latence DSR
(~2 ms — **piège** : ConPTY répond *lui-même* aux requêtes de position curseur,
cette mesure ne traverse jamais le parseur de Rio).

Le déblocage est venu de trois instruments :

1. **Sonde d'écho visuelle insensible au focus** : injection de touches par
   `PostMessage(WM_KEYDOWN)` + capture par `PrintWindow` + hash d'une
   sous-région, en boucle à ~40 ms. Verdict : codex réel = **206 ms** d'écho
   par frappe ; `cat` dans la même fenêtre = **0 ms**. Le *contenu* émis par
   l'app était donc la variable, pas le pipeline de Rio.
2. **Relais pty horodaté** entre Rio et l'app : la frappe atteint l'app en
   1-2 ms, l'app répond en ~20 ms → les ~185 ms manquants sont côté
   *rendu de la sortie* par Rio.
3. **Hôte ConPTY minimal instrumenté** (`CreatePseudoConsole` + lecture
   horodatée du pipe) : quand l'app enfant écrit `?2026h FRAME ?2026l` en un
   seul write, conhost **ré-émet la paire `?2026h?2026l` collée, PUIS la
   frame** dans des chunks suivants. ConPTY casse l'atomicité du synchronized
   output avant même que Rio ne voie le flux.

## 3. La cause et le correctif

Dans `rio-backend/src/performer/handler.rs` (héritage alacritty) :

- le dispatch CSI `('h', [b'?'])` **arme** `sync_state.timeout` (150 ms) quand
  le mode 2026 est activé ;
- le dispatch `('l', [b'?'])` ne le **désarmait pas** : l'hypothèse héritée
  était qu'un ESU n'est jamais parsé inline (après un BSU, les octets passent
  par `advance_sync`, qui gère l'ESU et nettoie). ConPTY viole cette
  hypothèse : la paire collée arrive `pending == false`, se parse inline, et
  le timeout reste armé. Tout chunk suivant est alors **bufferisé jusqu'au
  timeout de 150 ms** → ~150-200 ms par frappe. Les flux continus ne montrent
  rien (chaque paire suivante flush le buffer précédent) ; seule la sortie
  éparse — la frappe — paie.

Correctif (2 édits symétriques + 3 tests de régression) :

1. `('l', [b'?'])` : si le paramètre est 2026, `clear_timeout()` — miroir du
   cas `h`.
2. `stop_sync_internal`, branche `Some(bsu_offset)` : ré-armer le timeout —
   le replay du buffer peut contenir un ESU inline (désormais désarmant)
   alors qu'un *nouveau* BSU reste ouvert dans la queue du buffer.

## 4. Impact hors Windows

Le bug est **latent sur toutes les plateformes** : sur un pty Unix, une frame
complète (`h + frame + l` dans un seul read) se parse aussi inline → le
timeout reste collé une frame sur deux (parité chunk pair/impair). Effets :
toute sortie *non wrappée* qui suit peut être retardée jusqu'à 150 ms, et le
reader se réveille pour rien toutes les 150 ms tant que l'état est collé.
Rarement perceptible (les frames suivantes flushent), mais réel — le fix est
un correctif de justesse multi-plateforme, avec un gain spectaculaire sur
Windows et marginal ailleurs.
