---
title: "FCSC 2026 - Transport me"
date: 2026-04-14T10:52:57+02:00
draft: false
toc: true
images:
tags: 
  - FCSC 2026
  - FCSC
  - Intro
  - Hardware
  - Transport me
  - Write-Up
categories:
  - CTF
---

<div>
  <img src="/posts/images/logo_fcsc.svg" alt="Logo FCSC" style="width: 50%">
  </br>
</div>

Ceci est le write-up du challenge Intro Hardware **Transport me** du CTF [FCSC](https://fcsc.fr/) édition 2026 organisé par l'ANSSI.

---

## Énoncé

>Vous avez récupéré une carte de transport en commun, mais il ne reste que deux voyages dessus. Vous avez la possibilité d'interagir directement avec la carte. Pour obtenir le flag, vous devez réussir à voyager 5 fois.
>
>Pour interagir avec la carte, veuillez vous connecter au service en ligne indiqué.
>
><div>
>  <img src="/posts/fcsc2026_transport_me/images/card.png" alt="Card">
>  </br>
></div>
>
>`nc challenges.fcsc.fr 2300`

---

## Résolution

Notre objectif sur ce challenge est donc de voyager 5 fois avec une carte de transport n'ayant plus que 2 voyages.


### Investigation

L'énoncé ne nous donnant pas plus d'informations et étant donné qu'il n'y a pas de documentation, on va se connecter sur l'adresse du challenge pour essayer d'en apprendre plus sur son fonctionnement.

On tombe alors sur un menu:

```txt
0. Informations sur la carte
1. Voyage
2. Lecture UID
3. Lecture d'un bloc
4. Ecriture d'un bloc
5. Quitter
```

Et plus intéressant encore, le choix **0. Informations sur la carte**:

```txt
Les cartes du modèle FCSC-64B4 possèdent un identifiant (UID) de 32 bits, ainsi que quatre blocs de 16 bits réinscriptibles.

Le modèle mémoire est le suivant:
+---------------+----------------+------------------------+
| Adresse bloc  |    Contenu     |      Description       |
+---------------+----------------+------------------------+
|             0 | valeur 16 bits | Bloc 0                 |
|             1 | valeur 16 bits | Bloc 1                 |
|             2 | valeur 16 bits | Bloc 2                 |
|             3 | valeur 16 bits | Bloc 3                 |
|             4 | UID[15:0]      | Octets 1 et 0 de l'UID |
|             5 | UID[31:16]     | Octets 3 et 2 de l'UID |
+---------------+----------------+------------------------+

Les commandes exposées par la carte permettent de lire et d'écrire un bloc, ou bien de lire l'UID.
```

Ce sous-menu nous donne les caractéristiques de la carte, le plus intéressant ici est bien évidemment le tableau. On comprend qu'il s'agit d'une carte programmable simplifiée (normale car c'est un challenge d'intro) avec seulement **4 blocs** de 16 bits contenant de la donnée et **accessible en lecture et en écriture**.

### Identification du bloc compteur de voyage

En utilisant les commandes du menu, on va alors identifier le bloc comptant le nombre de voyage restant.

Pour cela, on va simplement les lire un par un, on cherche soit la valeur 2 (`00 02` car sur 16 bits) soit la valeur 65533 (`FF FD`).

{{< admonition type=note title="Note" >}}
La valeur `00 02` indiquerait explicitement le nombre de voyage restant la ou `FF FD` indiquerait le nombre de voyage maximum, il manquerait `2` pour arriver à `FF FD` (moins probable).
{{< /admonition >}}

Ce qui donne en résumé:

```txt
+---------------+----------------+
| Adresse bloc  |    Contenu     |
+---------------+----------------+
|             0 |          FF FF |
|             1 |          00 02 |
|             2 |          FF FF |
|             3 |          FF FF |
+---------------+----------------+
```

On voit clairement apparaître le **bloc 1** qui contient la valeur **00 02**.

### Ecriture du bloc 1

Maintenant qu'on a identifié le bloc contenant le nombre de voyages restants, on va pouvoir le modifier.

Pour cela, on va:
- Sélectionner le sous-menu **4. Ecriture d'un bloc**
- Préciser le **bloc 1**
- Ecrire une valeur égale ou supérieur à ``00 05``

Soit:
```txt
0. Informations sur la carte
1. Voyage
2. Lecture UID
3. Lecture d'un bloc
4. Ecriture d'un bloc
5. Quitter
>>> 4
Adresse du bloc à écrire: 1
Valeur à écrire, en hexadécimal (p.e., 'ffff'): FFFF
Ecriture effectuée
```

---

## Flag

Maintenant que l'on a rempli le compteur de voyages restants, on va pouvoir voyager 5 fois avec le sous-menu **1. Voyage**.

On voit alors le flag apparaître:

```txt
0. Informations sur la carte
1. Voyage
2. Lecture UID
3. Lecture d'un bloc
4. Ecriture d'un bloc
5. Quitter
>>> 1
Voyage effectué

0. Informations sur la carte
1. Voyage
2. Lecture UID
3. Lecture d'un bloc
4. Ecriture d'un bloc
5. Quitter
>>> 1
Voyage effectué

0. Informations sur la carte
1. Voyage
2. Lecture UID
3. Lecture d'un bloc
4. Ecriture d'un bloc
5. Quitter
>>> 1
Voyage effectué

0. Informations sur la carte
1. Voyage
2. Lecture UID
3. Lecture d'un bloc
4. Ecriture d'un bloc
5. Quitter
>>> 1
Voyage effectué

0. Informations sur la carte
1. Voyage
2. Lecture UID
3. Lecture d'un bloc
4. Ecriture d'un bloc
5. Quitter
>>> 1
Voyage effectué
Bravo, voilà le flag: FCSC{0519152bbd38a7a3c0fe98cbdb2f98d74a48598660718fe8b3695935ede5b37f}
```