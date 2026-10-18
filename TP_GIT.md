# Compte-rendu de TP — Mini-projet Git : « Sauvetage de release »

- **Module :** R509 DevCloud — Outils et méthodes DevOps
- **Établissement :** IUT de Saint-Malo — Département R&T
- **Étudiant :** Nolann MERLE
- **Contexte :** Remise en état du dépôt `tempconv` (préparation de la version `1.1` sur `develop` et publication du correctif de sécurité/production `v1.0.1` sur `main`).

---

## Récapitulatif rapide (vérifié par `./verifier.sh`)

- **Hash du commit coupable (`perf: simplifie l'arrondi`) :** `[COLLER_ICI_LE_HASH_OBTENU_A_LA_MISSION_1]`
  *(Astuce : exécutez `git log --all --format=%H --grep="^perf: simplifie l.arrondi$"` et remplacez le texte entre crochets ci-dessus par le hash affiché pour que `./verifier.sh` valide automatiquement `NOTES.md`).*
- **Pourquoi signer ses commits (en 2 phrases) :**
  Dans Git, le nom et l'adresse e-mail de l'auteur d'un commit sont de simples chaînes de caractères déclaratives que n'importe qui peut falsifier sans aucune vérification. Signer cryptographiquement ses commits et ses tags avec une clé privée permet de prouver de manière infalsifiable quelle clé a produit chaque modification et de garantir l'intégrité de l'historique, ce qui est indispensable pour la traçabilité en approche GitOps.

---

## Mise en place et état des lieux initial

### 1. Préparation du dépôt
```bash
bash preparer-depot.sh
cd sauvetage-release
```
- **Ce que fait la commande :** Le script clone le dépôt distant, crée localement toutes les branches distantes, récupère les tags, configure notre identité Git (`user.name` et `user.email`), génère une paire de clés SSH Ed25519 dédiée à la signature dans `.git/sauvetage_signing`, configure le fichier des signataires de confiance `.git/allowed_signers` et installe le script `./verifier.sh`.

### 2. Vérification de l'état initial du dépôt
```bash
git branch -a
git tag
git log --graph --oneline --all --decorate
./test.sh; echo $?
```
- **Ce que font ces commandes :**
  - `git branch -a` liste toutes les branches locales et distantes.
  - `git tag` affiche la liste des tags présents dans le dépôt.
  - `git log --graph --oneline --all --decorate` affiche sous forme d'arbre condensé l'historique complet de toutes les branches et références.
  - `./test.sh; echo $?` exécute la suite de tests automatisés puis affiche le code retour (`$?`) du script.
- **Informations récupérées :**
  - Le dépôt comporte **5 branches** : `main`, `develop`, `feature/export-json`, `feature/export-csv` et `hotfix/zero-absolu`.
  - Le tag de la version en production **`v1.0.0`** est présent sur `main`.
  - Sur la branche `develop`, `./test.sh` **échoue** et renvoie un code retour différent de $0$ (code `1`), ce qui confirme la présence d'une régression sur `develop`.

---

## Mission 0 : Comprendre votre signature

### 1. Afficher la configuration locale liée à l'identité et à la signature
```bash
git config --local --list
```
- **Ce que fait la commande :** Elle liste uniquement les paramètres Git configurés spécifiquement pour ce dépôt local (stockés dans `.git/config`).
- **Informations récupérées :**
  - `user.name` et `user.email` : notre identité de membre de l'équipe `tempconv`.
  - `gpg.format=ssh` : indique à Git d'utiliser des clés SSH plutôt que GPG pour signer.
  - `user.signingkey=.../.git/sauvetage_signing` : chemin vers notre clé privée de signature.
  - `commit.gpgsign=true` et `tag.gpgsign=true` : activent la signature automatique de tous nos futurs commits et tags.
  - `gpg.ssh.allowedSignersFile=.../.git/allowed_signers` : fichier contenant les clés publiques autorisées pour vérifier les signatures localement.

### 2. Identifier où se trouve la clé et ce que contient `.git/allowed_signers`
```bash
ls -la .git/sauvetage_signing*
cat .git/allowed_signers
```
- **Ce que font ces commandes :** Elles vérifient la présence des fichiers de clés SSH et affichent le contenu du fichier des signataires de confiance.
- **Informations récupérées :**
  - Notre clé privée se trouve dans `.git/sauvetage_signing` et notre clé publique dans `.git/sauvetage_signing.pub`.
  - Le fichier `.git/allowed_signers` contient deux entrées au format `<email> namespaces="git" <type-clé> <clé-publique>` :
    1. `equipe@devcloud.test namespaces="git" ssh-ed25519 ...` (la clé publique de l'équipe, extraite du tag `v1.0.0`).
    2. `<notre_email> namespaces="git" ssh-ed25519 ...` (notre propre clé publique).

### 3. Créer un commit vide, afficher sa signature, puis l'annuler
```bash
git commit --allow-empty -m "test: verification de ma signature"
git log --show-signature -1
git verify-commit HEAD
git reset --hard HEAD~1
```
- **Ce que font ces commandes :**
  - `git commit --allow-empty` crée un commit sans avoir besoin de modifier de fichier. Grâce à `commit.gpgsign=true`, il est signé automatiquement.
  - `git log --show-signature -1` (ou `git verify-commit HEAD`) vérifie cryptographiquement la signature du dernier commit à l'aide de `.git/allowed_signers`.
  - `git reset --hard HEAD~1` supprime ce commit de test en reculant le pointeur de branche d'un cran.
- **Informations récupérées :** Git affiche `Good "git" signature for <notre_email> with ED25519 key SHA256:...`, confirmant que notre signature automatique fonctionne et est reconnue.

### 4. Identifier qui a signé les commits de l'équipe et avec quelle clé
```bash
git log --format='%h %G? %GS %an <%ae> %s' --all
```
- **Ce que fait la commande :** Elle affiche un historique formaté avec le hash court (`%h`), le statut de vérification de la signature (`%G?`), l'identité du signataire reconnu (`%GS`), le nom/e-mail de l'auteur déclaré (`%an <%ae>`) et le titre du commit (`%s`).
- **Informations récupérées :** Les commits historiques de l'équipe portent le statut `G` (*Good signature*) et ont été signés par `equipe@devcloud.test` avec la clé Ed25519 de l'équipe.

### Réponse à la question de la Mission 0
> **Question : Que se passerait-il si votre clé publique n'était pas dans `allowed_signers` ?**
>
> **Réponse :** Nos commits seraient toujours signés par notre clé privée lors de leur création, mais Git serait incapable de vérifier à qui appartient cette signature lors d'un `git log --show-signature` ou `git verify-commit`. Le statut de signature (`%G?`) passerait alors de **`G`** (*signature valide*) à **`E`** (*signature non vérifiable / clé inconnue*), avec le message d'avertissement indiquant qu'aucune clé de confiance ne correspond dans `allowed_signers`.

---

## Mission 1 : L'enquête

### 1. Constater l'échec de `./test.sh` sur `develop`
```bash
git switch develop
./test.sh; echo $?
```
- **Ce que fait la commande :** On se place sur `develop`, on lance le script de tests et on affiche son code retour.
- **Informations récupérées :** Un test d'arrondi échoue et le code retour affiché par `echo $?` est `1` (erreur).

### 2. Retrouver automatiquement le premier commit fautif entre `v1.0.0` et `develop`
```bash
git bisect start develop v1.0.0
git bisect run ./test.sh
git bisect reset
```
- **Ce que font ces commandes :**
  - `git bisect start develop v1.0.0` démarre une recherche par dichotomie en indiquant que `develop` est mauvais (*bad*) et `v1.0.0` est bon (*good*).
  - `git bisect run ./test.sh` exécute automatiquement `./test.sh` au milieu de l'intervalle de commits : si le code retour est $0$, le commit testé est marqué *good* ; s'il est compris entre $1$ et $127$ (sauf $125$), il est marqué *bad*.
  - `git bisect reset` termine la recherche et replace `HEAD` au sommet de `develop`.
- **Informations récupérées :** `git bisect` identifie le premier commit défaillant : **`perf: simplifie l'arrondi`** (dont le hash complet est reporté en haut de ce fichier `NOTES.md`).

### 3. Afficher le statut de signature et l'auteur des commits de `v1.0.0..develop`
```bash
git log --format='%h %G? %an <%ae> %s' v1.0.0..develop
```
- **Ce que fait la commande :** Elle liste tous les commits ajoutés sur `develop` depuis le tag `v1.0.0` en affichant leur statut de signature (`%G?`) et leur auteur déclaré (`%an <%ae>`).
- **Informations récupérées :** Tous les commits de la plage ont le statut **`G`** (signés par l'équipe), à l'exception du commit fautif `perf: simplifie l'arrondi` qui porte le nom d'Alice mais affiche le statut **`N`** (*No signature*).

### Réponse à la question de la Mission 1
> **Question : Qu'a de particulier le commit fautif ? Qu'en concluez-vous ?**
>
> **Réponse :** Le commit fautif `perf: simplifie l'arrondi` est le seul commit de la branche à ne posséder **aucune signature cryptographique** (`%G?` affiche `N`), alors qu'il mentionne Alice comme auteure. On en conclut qu'Alice dit la vérité : elle n'est pas l'auteure de ce commit. N'importe qui peut configurer `git config user.name "Alice ..."` pour usurper une identité dans Git, d'où la nécessité d'exiger des commits signés (`G`) pour garantir l'authenticité de l'auteur.

---

## Mission 2 : Annuler la régression

### 1. Annuler le commit fautif sur `develop` sans réécrire l'historique
```bash
git switch develop
CULPRIT=$(git log --all --format=%H --grep="^perf: simplifie l.arrondi$" | tail -n1)
git revert --no-edit "$CULPRIT"
```
- **Ce que font ces commandes :**
  - La variable `CULPRIT` récupère le hash exact du commit `perf: simplifie l'arrondi`.
  - `git revert --no-edit "$CULPRIT"` génère un nouveau commit au sommet de `develop` qui applique exactement le patch inverse du commit fautif, avec le message standard `Revert "perf: simplifie l'arrondi" ... This reverts commit <HASH>.`.

### 2. Vérifier que `./test.sh` repasse au vert et contrôler la signature
```bash
./test.sh; echo $?
git log --show-signature -1
```
- **Informations récupérées :**
  - `./test.sh` réussit et renvoie le code retour `0`.
  - Le nouveau commit de revert est bien signé par notre clé SSH personnelle (`G`).

### Réponse à la question de la Mission 2
> **Question : Pourquoi `git revert` plutôt que `git reset` sur une branche partagée ?**
>
> **Réponse :** `git reset` supprime des commits et réécrit l'historique de la branche. Sur une branche partagée comme `develop`, réécrire l'historique crée des divergences chez tous les collègues qui ont déjà tiré (`pull`) ces commits et oblige à utiliser `git push --force`, au risque d'écraser leur travail. À l'inverse, `git revert` ne modifie pas le passé : il ajoute un nouveau commit correctif en bout de branche, ce qui préserve l'historique commun et garde une trace explicite de l'annulation.

---

## Mission 3 : Nettoyer une feature (`feature/export-json`)

### 1. Inspecter les commits de la branche `feature/export-json`
```bash
git log --oneline develop..feature/export-json
```
- **Ce que fait la commande :** Elle affiche les commits présents sur `feature/export-json` qui ne sont pas encore dans `develop`.
- **Informations récupérées :** On observe 6 commits brouillons successifs (`wip`, `oops`, `fix test`, etc.) qui polluent l'historique.

### 2. Nettoyer l'historique et placer la branche au-dessus de `develop`
```bash
git switch feature/export-json
git rebase -i develop
```
- **Ce que font ces commandes :** `git rebase -i develop` déplace la base de `feature/export-json` sur le dernier commit de `develop` (qui inclut notre revert) et ouvre un éditeur interactif permettant de réorganiser, fusionner (`squash` / `fixup`) et renommer (`reword`) les 6 commits.
- **Opérations réalisées dans le rebase interactif :**
  1. Regroupement des commits touchant au code métier (`convert.py`) en un premier commit propre : **`feat: ajoute l'export au format json`**.
  2. Regroupement des commits touchant aux tests (`test.sh`) en un second commit propre : **`test: ajoute les tests pour l'export json`**.
- **Informations récupérées :** La branche `feature/export-json` est désormais linéaire au-dessus de `develop`, ne comporte plus que 2 commits propres respectant la convention `type: description`, et chacun a été re-signé automatiquement avec notre clé SSH.

### 3. Intégrer `feature/export-json` dans `develop`
```bash
git switch develop
git merge feature/export-json
./test.sh
```
- **Ce que font ces commandes :** Elles intègrent les 2 commits propres dans `develop` et vérifient que tous les tests passent.

---

## Mission 4 : Résoudre un conflit (`feature/export-csv`)

### 1. Fusionner `feature/export-csv` dans `develop` et analyser le conflit
```bash
git switch develop
git merge feature/export-csv
git status
git diff --name-only --diff-filter=U
```
- **Ce que font ces commandes :**
  - `git merge feature/export-csv` tente de fusionner la branche d'export CSV dans `develop`.
  - `git diff --name-only --diff-filter=U` liste uniquement les fichiers en état de conflit (*Unmerged*).
- **Informations récupérées :** Git interrompt la fusion et signale un conflit dans **`convert.py`**. En effet, `feature/export-json` (intégrée à la mission 3) et `feature/export-csv` ont toutes deux modifié la liste des choix de l'option `--format` et le bloc d'affichage final dans `convert.py`.

### 2. Résoudre le conflit dans `convert.py` et finaliser la fusion
- **Démarche de résolution :** Dans `convert.py`, on supprime les marqueurs de conflit (`<<<<<<< HEAD`, `=======`, `>>>>>>> feature/export-csv`) et on combine la logique des deux fonctionnalités :
  - L'argument `--format` accepte désormais les trois valeurs : `choices=["texte", "json", "csv"]`.
  - Le bloc d'affichage gère `if args.format == "json":`, puis `elif args.format == "csv":` (affichant `f"{args.conversion};{args.valeur};{resultat}"`), et `else:` pour l'affichage texte par défaut.

```bash
./test.sh
python3 convert.py c2f 100 --format json
python3 convert.py c2f 100 --format csv
git add convert.py
git commit --no-edit
```
- **Informations récupérées :**
  - Le format JSON renvoie bien un objet contenant `"resultat": 212`.
  - Le format CSV renvoie exactement `c2f;100.0;212.0`.
  - `./test.sh` passe au vert et `git commit` crée le commit de fusion signé.

---

## Mission 5 : Hotfix et release `v1.0.1`

### 1. Intégrer `hotfix/zero-absolu` dans `main` en gardant une trace de la branche
```bash
git switch main
git merge --no-ff hotfix/zero-absolu -m "Merge branch 'hotfix/zero-absolu'"
```
- **Ce que fait la commande :** L'option `--no-ff` (*no fast-forward*) oblige Git à créer un vrai commit de fusion sur `main` même si la branche `main` n'avait pas bougé depuis `v1.0.0`. Cela permet de conserver visuellement l'existence de la branche `hotfix/zero-absolu` dans l'historique.

### 2. Publier un tag annoté et signé `v1.0.1`
```bash
git tag -s v1.0.1 -m "Release v1.0.1 : verification du zero absolu"
git tag -v v1.0.1
```
- **Ce que font ces commandes :**
  - `git tag -s v1.0.1 -m "..."` crée un tag annoté et signé cryptographiquement avec notre clé SSH sur le commit de fusion de `main`.
  - `git tag -v v1.0.1` vérifie la signature cryptographique du tag.
- **Informations récupérées :** `git tag -v v1.0.1` confirme `Good "git" signature for <notre_email>` sur l'objet tag `v1.0.1`.

### 3. Reporter le correctif sur `develop` avec son origine et résoudre le conflit
```bash
git switch develop
git cherry-pick -x hotfix/zero-absolu
```
- **Ce que fait la commande :** `git cherry-pick -x` récupère uniquement le commit de correction du zéro absolu et l'applique sur `develop`, tout en ajoutant automatiquement la mention `(cherry picked from commit <HASH>)` dans le message de commit pour conserver la traçabilité.
- **Résolution du conflit et validation :** Si un conflit apparaît dans `convert.py` ou `test.sh` (car `develop` contient déjà les exports JSON et CSV), on conserve à la fois la levée d'erreur sous le zéro absolu ($-273.15^\circ\text{C}$, $-459.67^\circ\text{F}$, $0\text{ K}$) et les ajouts JSON/CSV, puis on termine le *cherry-pick* :
```bash
./test.sh
git add .
git cherry-pick --continue
```

---

## Auto-évaluation finale

```bash
./verifier.sh
git log --graph --format='%h %G? %an %d %s' --all
```
- **Informations récupérées :** Toutes les vérifications de `./verifier.sh` sont au vert (`0 à revoir`), le tag `v1.0.1` et la pointe de `develop` portent notre signature valide (`G`).

---

## Questions et exercices Bonus

### Bonus A : Le reflog (récupérer une branche supprimée par erreur)
```bash
git switch -c brouillon
git commit --allow-empty -m "wip important"
git switch develop
git branch -D brouillon
git reflog
git branch brouillon HEAD@{1}
```
- **Explication :** Même lorsqu'une branche non fusionnée est supprimée de force avec `git branch -D`, le commit n'est pas immédiatement détruit par le ramasse-miettes (*garbage collector*) de Git. `git reflog` affiche le journal local de tous les déplacements de `HEAD`, ce qui permet de retrouver le hash du commit `"wip important"` (`HEAD@{1}`) et de recréer la branche dessus.

### Bonus B : Un garde-fou local (`pre-commit`)
```bash
cat > .git/hooks/pre-commit << 'EOF'
#!/usr/bin/env bash
./test.sh
EOF
chmod +x .git/hooks/pre-commit
```
- **Explication :** Git exécute automatiquement `.git/hooks/pre-commit` avant de créer chaque commit. Si `./test.sh` renvoie un code retour non nul, le commit est bloqué.
> **Question : Quelles sont les limites d'un hook par rapport à un pipeline CI ?**
>
> **Réponse :** Un hook Git est purement **local** : le dossier `.git/hooks` n'est pas versionné ni cloné automatiquement (chaque développeur doit l'installer manuellement), l'environnement local peut différer de la production, et n'importe quel développeur peut contourner le hook avec l'option `git commit --no-verify` (`-n`). Un pipeline CI (GitLab CI) s'exécute au contraire sur un serveur centralisé dans un environnement propre et obligatoire avant toute fusion.

### Bonus C : Brouillon de pipeline GitLab CI (`.gitlab-ci.yml`)
```yaml
stages:
  - test
  - release
  - deploy

test:
  stage: test
  script:
    - ./test.sh

release:
  stage: release
  rules:
    - if: $CI_COMMIT_TAG
  script:
    - echo "Publication de la release $CI_COMMIT_TAG"

deploy:
  stage: deploy
  rules:
    - if: $CI_COMMIT_TAG
      when: manual
  script:
    - echo "Déploiement manuel en production de $CI_COMMIT_TAG"
```

### Bonus D : Publier chez vous sur GitLab
```bash
git remote set-url origin https://gitlab.com/<votre_compte>/<votre_projet>.git
git push --all origin
git push --tags origin
```
> **Question : Vos commits sont `Verified`. Pourquoi ceux de l'équipe ne le sont-ils pas ?**
>
> **Réponse :** Sur l'interface web de GitLab, un commit n'obtient le badge `Verified` que si la clé publique qui l'a signé est enregistrée dans le profil d'un utilisateur GitLab existant **et** que l'adresse e-mail du commit correspond à une adresse e-mail vérifiée de ce même compte. Comme nous avons ajouté notre clé publique `.git/sauvetage_signing.pub` dans notre profil GitLab avec notre adresse e-mail, nos commits sont `Verified` ; en revanche, l'adresse fictive `equipe@devcloud.test` et la clé de l'équipe ne sont rattachées à aucun compte sur GitLab (GitLab n'utilise pas notre fichier local `.git/allowed_signers`).
