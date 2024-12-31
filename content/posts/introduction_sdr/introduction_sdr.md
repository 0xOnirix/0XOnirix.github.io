---
title: "Introduction à la SDR"
date: 2024-12-06T11:04:20+01:00
draft: false
toc: true
images:
tags: 
  - SDR
  - Gnuradio
  - GQRX
  - Radio
categories:
  - Radio
---

*"La SDR (pour Software Defined Radio) est un récepteur et éventuellement émetteur radio réalisés principalement par logiciel et dans une moindre mesure par matériel"* (merci [Wikipédia](https://fr.wikipedia.org/wiki/Radio_logicielle)).

Cet article est une introduction à la SDR orienté hacking, des principes de base au choix du matériel.

Je vais donc commencer par faire un court rappel sur ce que sont les ondes radio, la législation en vigeur, et enfin présenter les matériels et logiciels couramment utilisés.

---
## Les ondes radio

Une onde radioélectrique, ou **onde radio**, est une onde électromagnétique inférieure à 300 GHz, cela inclut les bandes FM, AM, VHF, les Takie-Walkie, les liaisons satellite, GSM, etc.

Ci dessous, un tableau donnant leur appelation en fonction de leur fréquence:

<figure align="center">
    <img alt="Fréquences radio" title="Fréquences radio" src="/posts/introduction_sdr/images/frequences_radio.png">
	<figcaption>Fréquences radio</figcaption>
</figure>

*Source: [Wikipedia](https://fr.wikipedia.org/wiki/Onde_radio) - Des spectres plus détaillés sont disponibles [ici](https://www.anfr.fr/fileadmin/medias/institutionnel/ANFR-spectre-frequences-juin-2020.pdf) et [ici](https://www.emitech.fr/en/radiofrequency-testing).*

Il est possible, avec un matériel adapté, de capter ces fréquences à moindre coût.

---
## Partie juridique

{{< admonition type=warning title="Attention !" >}}
Cette partie s'applique en France uniquement !
{{< /admonition >}}

Comme tout est réglementé, et l'émission d'ondes radio n'en étant pas excepté, il faut connaître la législation en vigueur pour ne pas faire n'importe quoi, il faut également garder en tête qu'elle peut changer d'un pays à un autre, aux abord des zones sensibles (zones militaires, aéroports, etc.) en même fonction des métropoles.

### Autorité

L'autorité qui s'occuper des ondes radio en France est l'Agence Nationale des Fréquences Radio (ANFR).

Elle a cinq missions principales:

- **Planifier** l'usage des fréquences
- **Gérer** les fréquences sites et données
- **Contrôler** le spectre
- **Protéger** les utilisateurs et services
- **Maîtriser** l'exposition du public aux ondes

Bien que l'Agence ne puisse pas directement donner d'amendes, elle peut enquêter et intervenir en qualité d'experte technique sur les cas d'usage repréhensibles des ondes radio (brouillage, empiétement de fréquence, radio pirate, etc.).

Elle travaille conjointement avec l'[ARCEP](https://www.arcep.fr) et l'[ARCOM](https://www.arcom.fr).


{{< admonition type=danger title="Attention !" >}}
La peine encouru en cas d'infraction peut aller jusqu'a 6 mois de prison et 30 000€ d'amende !
{{< /admonition >}}

### Réception

Pour ce qui est de la réception, il n'y a pas vraiment de restrictions, seul le déchiffrement de communications (notamment militaire ou du réseau [Tetra](https://fr.wikipedia.org/wiki/Terrestrial_Trunked_Radio)) est interdit.

Cependant, même en cas de simple captation de communications (chiffrées ou non), l'ANFR peut venir vérifier que vous ne faîtes rien d'illégale.

### Emission

Concernant l'émission, la règle générale est d'aller vérifier, en fonction de la bande de fréquences utilisée:
- Si on en a le **droit**, en prenant en compte la localisation
- Pendant combien de **temps**
- A quelle **puissance**

Car cela peut fortement varier d'une bande de fréquences à une autre.

Par exemple, pour le **LoRa** (fréquences de 4330.5MHz à 434.79MHz et 863MHz à 868.6MHz), le taux d'utilisation maximum est limité à **1%** du temps sur une heure glissante (en gros, pas plus de **36s** par heure) avec une puissance maximale de **25mW**.

*Des documents très complets sont disponibles sur le site de l'[ANFR](https://www.anfr.fr/planifier/le-tnrbf/lautorisation-dutilisation) et de l'[ARCEP](https://www.arcep.fr/la-regulation/grands-dossiers-reseaux-mobiles/le-guichet-start-up-et-innovation/le-portail-bandes-libres.html).*

---
## Le matériel

Afin de pouvoir écouter ces ondes, il faut pouvoir les capter, et pour ça, il existe beaucoup de matériels différents comme les clés **RTL-SDR**, le **Flipper Zero**, le **HackRF**, le **BladeRF**, le **Lime-SDR**, le **KrakenRF**, etc.

Concernant quoi choisir.. tout dépend de vos besoins et de votre budget, voici une liste non exhaustive des matériels couramment utilisés:

| **Hardware** | **RX/TX** | **Duplex** |              **Radio spectrum**             |   **Bandwidth**  |   **Price**   |
|:------------:|:---------:|:----------:|:-------------------------------------------:|:----------------:|:-------------:|
|  Clé RTL-SDR |     RX    |    Half    |              ~ 24MHz - 1.766GHz             |   6.7MHz - 8MHz  |     ~ 30€     |
| Flipper Zero |   RX/TX   |    Half    | 300MHz-348MHz, 387MHz-464MHz, 779MHz-928MHz |  58KHz - 812KHz  |     ~ 200€    |
|    HackRF    |   RX/TX   |    Half    |                 30MHz - 6GHz                |       20MHz      |     ~ 320€    |
|    BladeRF   |   RX/TX   |    Full    |               300MHz - 3.8GHz               |       28MHz      |     ~ 780€    |
|     USRP     |   RX/TX   |    Full    |                 50MHz - 6GHz                | 16MHz - 61.44MHz |     ~ 2k€     |
|   Lime-SDR   |   RX/TX   |    Full    |               ~ 30MHz - 3.8GHz              | ~ 40MHz - 120MHz | ~ 340€ - 20k€ |
|   KrakenSDR  |   RX/TX   |    Full    |               24MHz - 1766MHz               |      2.56MHz     |     ~ 500€    |


Comme on peut le voir, chaque solution à ses avantages et inconvenients. 

Si vous débutez dans le monde de la SDR, je ne peux que vous conseiller la clé RTL-SDR car ça coute moins de 30€, ça à une plage d'utilisation très large, une antenne est fournie avec, c'est petit et compatible avec tout !

Il y en a ici aussi une pléthore, je vous conseille cependant d'utiliser une **Clé USB RTL2832 avec tuner R860** (disponible sur [Passion radio](https://www.passion-radio.fr/recepteurs-sdr/rtl-sdr-r820t2-248.html) et [Amazon](https://www.amazon.fr/R%C3%A9cepteur-ventouse-adaptateur-femelle-r%C3%A9cepteur/dp/B094ZJ3VVN/ref=sr_1_4?__mk_fr_FR=%C3%85M%C3%85%C5%BD%C3%95%C3%91&crid=J5HBVVQTMDV6&keywords=Cl%C3%A9+USB+RTL2832&qid=1687622893&sprefix=cl%C3%A9+usb+rtl2832%2Caps%2C223&sr=8-4)), elle à un spectre assez large, de 24MHz à 1,766GHz, et posséde le chipset **Realtek RTL2832U** ainsi que le tuner **Rafael Micro R860** qui sont faits pour la SDR.

---
## Les outils logiciels

Voici grossiérement la liste des logiciels les plus utilisés:

- [GNURadio](https://github.com/gnuradio/gnuradio)
- [GQRX](https://github.com/gqrx-sdr/gqrx) (ou [Airspy](https://airspy.com/) pour Windows)
- [Dump1090](https://github.com/antirez/dump1090)

Ils sont open-source (excepté Airspy) et disponibles sur Linux et Windows.

**GQRX** (ou **Airpsy**) permettent d'analyser un spectre, de le démoduler selon certaines modulations et de l'écouter/enregistrer.

**Dump1090** est spécifique à la récupération des trames des balises d'avions (**ADS-B**), il permet de visualiser les informations qu'elles contiennent et afficher les appareils sur une carte.

Enfin, **GnuRadio** est LE logiciel de SDR, il permet de faire littéralement ce qu'on veut, en réception ou en émission !

---
## Conclusion

Voila qui clos cette introduction à la SDR, vous savez maintenant à peu près de quoi il s'agit, la réglementation en vigeur, le matériel nécessaire ainsi que les logiciels couramment utilisés.

Dans un prochain chapitre, nous allons voir comment utiliser ces logiciels ainsi que quelques exemples concrets.

---
## Sources

https://fr.wikipedia.org/wiki/Onde_radio

https://anfr.fr

https://www.radioamateurs-france.fr/wp-content/uploads/2015/07/A-ANFR-presentation.pdf

https://arcep.fr

https://arcom.fr

https://fr.wikipedia.org/wiki/Terrestrial_Trunked_Radio

http://www.taylorkillian.com/2013/08/sdr-showdown-hackrf-vs-bladerf-vs-usrp.html

https://docs.flipper.net/

https://limemicro.com/

https://www.krakenrf.com/

https://github.com/krakenrf/krakensdr_docs/wiki#technical-specifications

https://www.passion-radio.org/blog/sdr-sharp-dongle-rtl2832-r820t-e4000/76466

https://homepages.uni-regensburg.de/~erc24492/SDR/Data_rtl2832u.pdf

https://github.com/gnuradio/gnuradio

https://github.com/gqrx-sdr/gqrx

https://airspy.com/

https://github.com/antirez/dump1090
