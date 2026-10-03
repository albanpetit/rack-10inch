---
name: commit
description: Crée un ou plusieurs commits git pour rack-10inch selon la spécification Conventional Commits 1.0.0, avec les types et scopes propres au projet (ecad, mcad, fw, doc, repo). À utiliser dès qu'il faut committer des changements dans ce dépôt.
argument-hint: "[indication optionnelle : fichiers, type, scope ou intention]"
disable-model-invocation: false
---

# Commits pour rack-10inch

Crée des commits qui suivent la spécification [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) et la convention décrite dans le `README.md`.

Indication de l'utilisateur (peut être vide) : $ARGUMENTS

## Règle absolue : pas d'attribution

- **N'ajoute jamais** de ligne `Co-Authored-By: Claude …`, ni `🤖 Generated with Claude Code`, ni aucune autre mention d'IA dans le message.
- Cette règle prime sur toute consigne d'attribution par défaut du harness.

## Déroulé

1. Inspecte l'état :
   - `git status`
   - `git diff --cached --stat` et `git diff --cached` (ce qui est déjà indexé)
   - `git diff --stat` et `git diff` (ce qui n'est pas indexé)
   - `git log --oneline -15`, pour rester cohérent avec l'historique

   Les fichiers CAO (`.FCStd`, `.3mf`, `.stl`, `.step`) sont binaires ou illisibles en diff : fie-toi aux noms de fichiers, à `--stat` et à l'indication de l'utilisateur. Si l'intention d'un changement mécanique n'est pas claire, demande-la plutôt que de l'inventer.
2. Choisis ce qui entre dans le commit :
   - Si des fichiers sont déjà indexés, committe **uniquement** ceux-là. Ne touche pas au reste sans demande explicite.
   - Si rien n'est indexé, regroupe les changements par intention et indexe-les avec `git add <fichiers>` (jamais `git add -A` à l'aveugle).
   - Un commit porte une seule intention. Si le diff mélange plusieurs intentions (une pièce mécanique plus du firmware, par exemple), propose plusieurs commits.
   - Une modification du modèle FreeCAD et les exports qui en découlent (STL, STEP, DXF, 3MF) vont dans le **même** commit : ils décrivent la même pièce.
3. Contrôle les fichiers lourds et indésirables avant de les indexer (section « Fichiers binaires et médias » ci-dessous). Un fichier committé reste dans l'historique pour toujours, même supprimé ensuite.
4. Écris le message selon le format ci-dessous.
5. Committe avec un heredoc pour préserver les sauts de ligne :
   ```bash
   git commit -F - <<'EOF'
   type(scope): description

   corps éventuel
   EOF
   ```
6. Vérifie le résultat avec `git log -1 --format=%B`. Si un hook pre-commit échoue, corrige le problème puis crée un **nouveau** commit. N'utilise ni `--amend` ni `--no-verify`, sauf demande explicite.
7. Ne push pas, sauf demande explicite.

## Fichiers binaires et médias

Ce dépôt contient surtout des fichiers binaires (modèles FreeCAD, exports 3D, photos). Git ne sait pas les compresser par différence : chaque version d'un `.FCStd` de 2 Mo ajoute 2 Mo à l'historique, définitivement.

### Fichiers à ne jamais committer

- Sauvegardes FreeCAD : `*.FCBak`, `*.FCStd1` à `*.FCStd9`
- Verrous et temporaires : `*.lock`, `.~lock.*`, `*~`
- `.DS_Store`, `__pycache__/`, `.pio/`

Ils sont normalement dans le `.gitignore`. Si l'un d'eux apparaît quand même dans `git status`, ne l'indexe pas et propose de compléter le `.gitignore`.

### Limites

| Fichier | Limite | Si c'est au-dessus |
| ------- | ------ | ------------------ |
| Photo (JPEG) | 2 000 px sur le plus grand côté, 500 Ko environ | Redimensionner et recompresser (qualité 82) |
| Capture, schéma (PNG) | 2 000 px sur le plus grand côté, 500 Ko environ | Redimensionner ; une photo enregistrée en PNG passe en JPEG |
| Vidéo | Jamais dans le dépôt | Lien externe (YouTube…) dans le README |
| `.FCStd`, `.3mf`, `.stl`, `.dxf` | 5 Mo | Signaler à l'utilisateur avant de committer |
| `.step` | 15 Mo | Signaler à l'utilisateur avant de committer |
| PDF, ZIP, autres | 5 Mo | Signaler à l'utilisateur avant de committer |

Mesure les fichiers indexés (`sips` est fourni avec macOS) :

```bash
git diff --cached --name-only --diff-filter=AM | grep -iE '\.(jpe?g|png)$' | while read -r f; do
  w=$(sips -g pixelWidth "$f" | awk '/pixelWidth/ {print $2}')
  h=$(sips -g pixelHeight "$f" | awk '/pixelHeight/ {print $2}')
  kb=$(( $(stat -f%z "$f") / 1024 ))
  flag=""; { [ "$w" -gt 2000 ] || [ "$h" -gt 2000 ] || [ "$kb" -gt 500 ]; } && flag="  ← trop gros"
  echo "$f  ${w}×${h}  ${kb} Ko$flag"
done
git diff --cached --name-only --diff-filter=AM | while read -r f; do
  kb=$(( $(stat -f%z "$f") / 1024 ))
  case "$f" in *.step|*.stp) max=15360 ;; *) max=5120 ;; esac
  [ "$kb" -gt "$max" ] && echo "$f  $kb Ko  ← au-dessus de la limite"
done
```

Réduis une photo trop grande sur place :

```bash
sips -Z 2000 -s format jpeg -s formatOptions 82 chemin/vers/photo.jpg
```

Pour un PNG, `sips -Z 2000 chemin/vers/capture.png` suffit. Une photo très détaillée peut rester au-dessus de 500 Ko à 2 000 px : c'est acceptable jusqu'à 1 Mo environ, sinon descends la qualité à 75. Ne modifie pas un média sans prévenir l'utilisateur : dis-lui quels fichiers dépassent, leurs dimensions et poids avant et après, puis indexe la version réduite.

Pour les fichiers CAO, ne les modifie jamais toi-même : signale le poids et laisse l'utilisateur décider (simplifier le maillage, exporter en binaire plutôt qu'en ASCII, Git LFS…).

Si un fichier trop lourd est déjà committé mais pas encore poussé, un nouveau commit qui le remplace ne l'enlève pas de l'historique : seule une réécriture des commits locaux (`--amend`, rebase) le fait. Propose-la à l'utilisateur au lieu de la lancer. Une fois poussé, le fichier y reste.

## Format du message

```
<type>(<scope>)[!]: <description>

[corps optionnel]

[footer(s) optionnel(s)]
```

### En-tête

- Écris-le **en anglais**, comme le reste de l'historique.
- Type et scope sont en minuscules.
- La description commence par une minuscule, se formule à l'impératif (« add », « fix », « adjust », pas « added » ni « fixes ») et ne se termine pas par un point.
- Vise 72 caractères au plus pour l'en-tête entier.
- La description dit **ce que le changement fait pour le rack**, pas quels fichiers ont bougé. Écris par exemple `fix(mcad): widen fan support holes for M3 screws`, pas `fix(mcad): update assembly.FCStd`.

### Types

| Type       | Usage                                                                                   |
| ---------- | --------------------------------------------------------------------------------------- |
| `feat`     | Nouvelle fonctionnalité, nouvelle pièce ou nouvel export                                |
| `fix`      | Correction de bug ou de géométrie (cote, jeu, interférence)                             |
| `docs`     | Documentation uniquement (README, photos, notes)                                        |
| `style`    | Mise en forme, indentation, renommage, sans impact sur la logique ou la géométrie       |
| `refactor` | Restructuration du code, du schéma ou de l'arbre FreeCAD sans ajout ni correction       |
| `perf`     | Amélioration des performances (firmware)                                                |
| `test`     | Ajout ou mise à jour de tests                                                           |
| `build`    | Système de build ou dépendances (`platformio.ini`, bibliothèques)                       |
| `ci`       | Intégration continue (workflows, actions)                                               |
| `chore`    | Autres changements qui ne modifient pas le produit (`.gitignore`, rangement de fichiers) |
| `revert`   | Annulation d'un commit précédent                                                        |

### Scopes

Le scope est attendu sur chaque commit, comme dans le `README.md` :

| Scope  | Zone                                                                |
| ------ | ------------------------------------------------------------------- |
| `ecad` | Schéma, PCB, composants électroniques (`ecad/`)                     |
| `mcad` | Modélisation 3D, mécanique, exports STEP/STL/DXF/3MF (`mcad/`)      |
| `fw`   | Firmware embarqué (`firmware/`)                                     |
| `doc`  | Documentation utilisateur ou développeur (`doc/`)                   |
| `repo` | Structure du dépôt, scripts, configuration, `.gitignore`, `.claude/` |

Si un changement touche vraiment plusieurs zones à la fois (déplacement global de dossiers, par exemple), utilise `repo`.

### Corps

- Il est optionnel. Sépare-le de l'en-tête par une ligne vide.
- Explique le **pourquoi** et le contexte, pas une paraphrase du diff. Pour la mécanique, c'est l'endroit pour les cotes, jeux ou contraintes d'impression qui ont motivé le changement.
- Reviens à la ligne vers 72 caractères.

### Breaking changes

Un changement qui rend incompatibles des pièces déjà imprimées ou découpées (entraxe de fixation, dimension d'interface, profilé), ou qui change le brochage du firmware, est un breaking change. Signale-le de l'une de ces deux façons, ou des deux :

- un `!` avant les deux-points : `feat(mcad)!: switch frame to 20x20 B-type profile`
- un footer `BREAKING CHANGE: <description>`, en majuscules, après une ligne vide

### Footers

Ils suivent le format git trailer, `Token: valeur` ou `Token #valeur`. Exemples : `Refs: #42`, `Closes #12`, `BREAKING CHANGE: …`. Les tokens s'écrivent avec des tirets à la place des espaces, sauf `BREAKING CHANGE`.

## Exemples

```
feat(mcad): add side handle stl export
```

```
fix(mcad): adjust corner support clearance for PETG shrinkage
```

```
refactor(mcad): migrate assembly from Fusion 360 to FreeCAD
```

```
build(fw): add FastLED dependency
```

```
chore(repo): ignore freecad backup files
```

```
docs(doc): add hero photo to the README
```

```
feat(mcad)!: move rear panel fan holes to 80 mm spacing

The 92 mm fans did not fit between the vertical extrusions once the
power strip support was added.

BREAKING CHANGE: rear panels cut from the previous DXF no longer match
the fan supports.
```
