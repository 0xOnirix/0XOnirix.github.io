---
title: "Introduction à la SDR 1/2"
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

<figure align="center">
    <img alt="Bandes de fréquence radio" title="Bandes de fréquence radio" src="/posts/introduction_sdr/images/bands-radio-frequency-spectrum.jpg">
	<figcaption>Bandes de fréquence radio - <a hreaf="https://www.britannica.com">Britannica</a></figcaption>
</figure>

Ces ondes sont définit par 3 caractéristiques principales:

- La phase
- L'amplitude
- La période (1/fréquence)

<figure align="center">
    <img alt="Phase, Amplitude et Période" title="Phase, Amplitude et Période" src="/posts/introduction_sdr/images/amplitude_phase_period.svg">
	<figcaption>Phase, Amplitude et Période d'un signal - <a href="https://pysdr.org/content/frequency_domain.html">PySDR</a></figcaption>
</figure>

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
1. Si on en a le **droit**, en prenant en compte la localisation
2. Pendant combien de **temps**
3. A quelle **puissance**

Car cela peut fortement varier d'une bande de fréquences à une autre.

Par exemple, pour le **LoRa** (fréquences de 4330.5MHz à 434.79MHz et 863MHz à 868.6MHz), le taux d'utilisation maximum est limité à **1%** du temps sur une heure glissante (en gros, pas plus de **36s** par heure) avec une puissance maximale de **25mW**.

*Des documents très complets sont disponibles sur le site de l'[ANFR](https://www.anfr.fr/planifier/le-tnrbf/lautorisation-dutilisation) et de l'[ARCEP](https://www.arcep.fr/la-regulation/grands-dossiers-reseaux-mobiles/le-guichet-start-up-et-innovation/le-portail-bandes-libres.html).*

---
## Le matériel

Afin de pouvoir écouter ces ondes, il faut pouvoir les capter, et pour ça, il existe beaucoup de matériels différents avec chacun leurs spécificités.

Je vais d'abord expliquer les spécificités les plus importantes avant de présenter les solutions les plus connus/utilisées.

{{< admonition type=warning title="Attention !" >}}
Il n'existe pas de solution universelle, chaque produit à ses avantages et inconvenients, il faut le choisir en fonction des besoins du projet.
{{< /admonition >}}

### Spectre radio

Le spectre d'un produit est la gamme de fréquence qu'il est capable d'écouter.

Par exemple, pour du WiFi 6, c'est aux alentours de **2,4GHz** et **5GHz**.

Pour du NFC, c'est la fréquence **13,56MHz**.

### Bande passante

La bande passante définit la quantité d'information que peut transmettre un signal à un temps T, plus elle est grande mieux c'est mais plus ça sera chère.

Pour avoir un ordre de grandeur, le WiFi 3 (2,4GHz) à une bande passante de **20MHz** et un débit max de **54Mb/s** tandis que le GSM (~ 900MHz) à une bande passante de **200kHz** et un débit max de **271kb/s**.

### Duplex

Un duplex est un canal de communication, on parle de **full-fuplex** quand on peut transmettre et recevoir en même temps et de **half-duplex** quand ce n'est pas le cas.

Un cas concret d'utilisation du half-duplex est le MITM (WiFi, GSM, etc) car on doit faire la passerelle entre l'AP et l'appareil victime en temps réel.

### RX/TX

RX signifie **réception** et TX **Transmission**, il faut juste garder en tête que tous les matériels ne peuvent pas transmettre.

### Matériel connus

De la **clé RTL-SDR**, en passant par le **Flipper Zero**, le **HackRF**, le **BladeRF**, le **Lime-SDR**, le **KrakenRF**, etc. Il existe une pléthore de matériel SDR, il y en aura donc forcémment un qui vous conviendra !

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

Il y en a ici aussi beaucoup, je vous conseille cependant d'utiliser une **Clé USB RTL2832 avec tuner R860** (disponible sur [Passion radio](https://www.passion-radio.fr/recepteurs-sdr/rtl-sdr-r820t2-248.html) et [Amazon](https://www.amazon.fr/R%C3%A9cepteur-ventouse-adaptateur-femelle-r%C3%A9cepteur/dp/B094ZJ3VVN/ref=sr_1_4?__mk_fr_FR=%C3%85M%C3%85%C5%BD%C3%95%C3%91&crid=J5HBVVQTMDV6&keywords=Cl%C3%A9+USB+RTL2832&qid=1687622893&sprefix=cl%C3%A9+usb+rtl2832%2Caps%2C223&sr=8-4)), elle à un spectre assez large, de 24MHz à 1,766GHz, et posséde le chipset **Realtek RTL2832U** ainsi que le tuner **Rafael Micro R860** qui sont faits pour la SDR.

---
## Les outils logiciels

Voici grossiérement la liste des logiciels les plus utilisés:

- [GNURadio](https://github.com/gnuradio/gnuradio)
- [Universal Radio Hacker (URH)](https://github.com/jopohl/urh)
- [Inspectrum](https://github.com/miek/inspectrum)
- [GQRX](https://github.com/gqrx-sdr/gqrx) (ou [Airspy](https://airspy.com/) pour Windows)
- [Dump1090](https://github.com/antirez/dump1090)

Ils sont open-source (excepté Airspy) et disponibles sur Linux et Windows.

**GQRX** (ou **Airpsy**) permettent d'analyser un spectre, de le démoduler selon certaines modulations et de l'écouter/enregistrer.

**Inspectrum** est un outils d'analyse de signaux déjà capturés.

**Universal Radio Hacker (URH)** est une suite complète pour les protocoles sans fils.

**Dump1090** est spécifique à la récupération des trames des balises d'avions (**ADS-B**), il permet de visualiser les informations qu'elles contiennent et afficher les appareils sur une carte.

Enfin, **GnuRadio** est LE logiciel de SDR, il permet de tout faire mais il est très bas niveau.

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

https://resources.altium.com/fr/p/whats-difference-between-data-rate-and-bandwidth

https://pysdr.org/

https://fr.wikipedia.org/wiki/D%C3%A9bits_et_port%C3%A9es

https://fr.wikipedia.org/wiki/IEEE_802.11n

https://fr.wikipedia.org/wiki/Global_System_for_Mobile_Communications

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
