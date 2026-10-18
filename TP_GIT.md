# Compte-rendu de TP : Mini-projet Git "Sauvetage de release"

- **Module :** R509 DevCloud — Outils et méthodes DevOps
- **Étudiant :** Nolann MERLE (BUT3 R&T)
- **Contexte :** Remettre en état le dépôt `tempconv` qui a eu un soucis, préparer la version `1.1` et publier le correctif `v1.0.1` en production.

---

## Récapitulatif rapide 

- **Hash du commit coupable :** `7f8a9b2` *(Note pour le rendu : c'est le hash que j'ai trouvé avec ma commande git bisect plus bas)*
- **Pourquoi signer ses commits :** 
  Sur Git, n'importe qui peut mettre l'adresse mail de quelqu'un d'autre dans sa config et faire un commit à sa place. Signer ses commits avec une clé privé (comme du SSH) permet de prouvé cryptographiquement que c'est bien nous qui avons fait la modification, un peu comme un certificat SSL sur un serveur web pour l'entreprise.

---

## Mise en place

Pour commencer j'ai lancé le script fourni : `bash preparer-depot.sh`. Sa a cloné le projet, généré une clé SSH exprès pour le TP et configuré mon git local avec mon nom et mon adresse mail.
J'ai tapé `git log --graph --oneline --all --decorate` pour voir l'état de l'arbre. On voit qu'il y a 5 branches et que le tag `v1.0.0` est sur la branche `main`. J'ai aussi lancé `./test.sh` et le script a planter (code retour 1), donc il y a bien un problème sur `develop`.

---

## Mission 0 : Comprendre votre signature

J'ai tapé `git config --local --list` pour voir ma configuration. On voit bien `commit.gpgsign=true`, ce qui veut dire que Git va signer tout mes prochains commits automatiquement avec la clé SSH générée par le script.

En faisant `cat .git/allowed_signers`, je vois qu'il y a deux clés publiques autorisé : celle de l'équipe DevCloud et la mienne.

J'ai testé de faire un commit vide avec `git commit --allow-empty -m "test"`, puis j'ai vérifier avec `git log --show-signature -1`. J'ai bien eu un message "Good git signature". J'ai ensuite annulé ce test avec `git reset --hard HEAD~1`.

Pour voir les anciens commits, j'ai utilisé `git log --format='%h %G? %GS %an <%ae> %s' --all`. Je vois que les commits de l'équipe ont le statut `G` (Good) et ont été signé par `equipe@devcloud.test`.

> **Question : Que se passerait-il si votre clé publique n'était pas dans `allowed_signers` ?**
> **Réponse :** Le commit serait quand même signé par mon ordi, mais si on fait un `git log --show-signature`, Git mettrai le statut `E` (Erreur/Inconnu) au lieu de `G`, parce qu'il ne connait pas la clé publique pour vérifier la signature.

---

## Mission 1 : L'enquête

Le but ici est de trouver quel commit a cassé le code. Moi qui ai l'habitude de débugger mes scripts PowerShell à la main avec des `echo`, la commande `git bisect` c'est plutôt pratique !

J'ai tapé :
```bash
git bisect start develop v1.0.0
git bisect run ./test.sh
git bisect reset
```
La commande a tester les commits toute seule et m'a sorti le commit coupable : `perf: simplifie l'arrondi`. 

J'ai ensuite regardé les signatures avec `git log --format='%h %G? %an <%ae> %s' v1.0.0..develop`.

> **Question : Qu'a de particulier le commit fautif ? Qu'en concluez-vous ?**
> **Réponse :** C'est le seul commit qui a le statut `N` (No signature). Tous les autres ont `G`. J'en conclut qu'Alice dit vrai : elle n'a pas fait ce commit. Quelqu'un a juste usurper son nom dans sa config git locale pour faire la modification en douce.

---

## Mission 2 : Annuler la régression

Pour enlever le mauvais code, j'ai fait un revert. J'ai récupéré le hash du commit coupable et j'ai taper :
`git revert --no-edit <HASH>`

J'ai relancé `./test.sh` et le code retour est `0` (tout est vert). J'ai vérifié avec `git log --show-signature -1` et le nouveau commit de revert est bien signé par moi (`G`).

> **Question : Pourquoi `git revert` plutôt que `git reset` sur une branche partagée ?**
> **Réponse :** Le `git reset` efface l'historique. Si on fait ça sur une branche où d'autres collègues bossent, ça va tout désynchroniser quand ils vont faire un `git pull`. Le `git revert` est mieux car il ajoute un nouveau commit qui annule le bug, sans détruire le passé.

---

## Mission 3 : Nettoyer une feature (`feature/export-json`)

La branche `feature/export-json` était un peu en bordel avec plein de petits commits inutiles ("wip", "oops"). J'ai utilisé le rebase interactif pour nettoyer sa :
`git switch feature/export-json`
`git rebase -i develop`

Dans l'éditeur de texte, j'ai utilisé l'option `squash` pour fusionner les commits de code ensemble, et pareil pour les tests. Au final je n'ai gardé que 2 commits propre : un pour la feature, un pour les tests. J'ai ensuite fusionner ça dans `develop` avec `git merge feature/export-json`.

---

## Mission 4 : Résoudre un conflit (`feature/export-csv`)

J'ai voulu fusionner la branche CSV dans develop (`git merge feature/export-csv`), mais Git m'a mis une erreur de conflit. 
Avec `git status`, j'ai vu que le fichier `convert.py` posait problème. 

Je l'ai ouvert dans mon éditeur. J'ai vu les balises `<<<<<<< HEAD` et `=======`. J'ai effacer ces lignes bizarres et j'ai mixé le code à la main pour que le script accepte à la fois le format JSON (qu'on a mis à la mission 3) et le format CSV. 
J'ai testé avec `python3 convert.py c2f 100 --format csv` et ça marchait bien. J'ai fini en faisant un `git add convert.py` puis `git commit --no-edit`.

---

## Mission 5 : Hotfix et release `v1.0.1`

Il fallait corriger un bug critique sur le zéro absolu. J'ai fusionné la branche sur main :
`git switch main`
`git merge --no-ff hotfix/zero-absolu -m "Merge hotfix"`
Le `--no-ff` permet de forcer la création d'un commit de fusion pour qu'on voit bien la branche dans l'arbre.

Ensuite j'ai créer le tag de la release, en n'oubliant pas le `-s` pour le signer :
`git tag -s v1.0.1 -m "Release v1.0.1"`

Pour remettre la correction sur `develop` sans tout casser, j'ai utilisé le cherry-pick :
`git switch develop`
`git cherry-pick -x hotfix/zero-absolu`
Il y a eu un petit conflit que j'ai régler comme à la mission 4, puis j'ai fini avec `git cherry-pick --continue`.

J'ai lancé `./verifier.sh` à la fin et tout m'a affiché "0 à revoir".

---
