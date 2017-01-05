# Guide d'utilisation de Git/GitHub

Ce guide ne vise pas à être exaustif sur l'utilisation de Git et GitHub. Il ne reprend que quelques principes de base et renvoie vers des ressources qui nous semblent intéressantes/utiles.

Si nécessaire n'hésitez pas à vous repporter à la documentation officielle: https://git-scm.com/doc

**IMPORTANT: une fois publiée sur GitHub, il est généralement impossible de supprimer complètement une information car elle reste dans l'historique des modifications. Attention de ne pas transmettre via Git des données confidentielles (ex. mot de passe). Il est possible d'utiliser le fichier `.gitigore` pour éviter ces maladresses.** 


## Installation de Git

La première chose à faire: installer Git.
Cf. https://git-scm.com/


## Publier des fichiers dans un entrepôt

Se placer dans le répertoire souhaité:

```
$ cd mon_repertoire
```

Cloner le dépôt (ici le dépôt documentation):

```
$ git clone https://github.com/cigalsace/documentation.git
```

Apporter les modification souhaitées:
- Créer les dossiers et fichiers dans l'arborescence clonées (via l'explorateur de fichiers).
- Modifier les fichiers existants
- etc.

Ajouter les fichiers modifiés dans la file d'attente de Git

```
$ git add .
```

Créer un commit en spécifiant le message de description de la modification apportée (ne pas être trop long: 50 caratères):

```
$ git commit -m "Mon super message de description de ma super modification"
```

Pousser le commit sur GitHub (indiquer login et mot de passe si demandé):

```
$ git push
```

Résultat:

![resultat.jpg](resultat.jpg)


## Fichier `.gitignore`

Le fichier `.gitignore` permet de spécifier des fichiers qui ne seront pas poussés sur GitHub lors de la publication de code.
Cf. https://git-scm.com/docs/gitignore