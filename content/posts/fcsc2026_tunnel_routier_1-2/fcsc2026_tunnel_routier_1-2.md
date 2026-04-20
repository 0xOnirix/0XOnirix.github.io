---
title: "FCSC 2026 - Tunnel Routier 1/2"
date: 2026-04-12T12:00:00+00:00
draft: false
toc: false
images:
tags: 
  - FCSC 2026
  - FCSC
  - Misc
  - Tunnel Routier 1/2
  - Write-Up
categories:
  - CTF
---

<div>
  <img src="/posts/fcsc2026_tunnel_routier_1-2/images/logo_fcsc.svg" alt="Logo_FCSC" style="width: 50%">
  </br>
</div>

Ceci est le write-up de la première partie du challenge [Misc](/tags/misc/) **Tunnel Routier** du CTF **France CyberSecurity Challenge** ([FCSC](https://fcsc.fr/)) édition 2026 organisé par l'ANSSI.

## Énoncé

>Dans cette première partie de l'épreuve, vous devez retrouver un token caché dans les données d'identification de l'automate pour vous permettre d'accéder à la deuxième partie de l'épreuve.
>
>Note: La description du système est disponible sur une [page dédiée](https://fcsc.fr/tunnel-routier-76bea48861178b0c)
>
>`nc tunnel-routier.fcsc.fr 502`

### Documentation

Voici la documentation présente à l'adresse: https://fcsc.fr/tunnel-routier-76bea48861178b0c


**Tunnel routier : documentation**

Le tunnel de Grenelle d'une longueur de 5km doit être inauguré ce jour, pour cela des responsables locaux ont fait le déplacement pour assister à sa première mise en service. Le système de contrôle-commande de ce tunnel est assuré par un automate programmable industriel (PLC) communiquant avec le SCADA au travers d'un protocole industriel sur le port TCP/502. Ce système permet de réguler des valeurs de process offrant une traversée du tunnel en toute sécurité pour les usagers. Les valeurs de process régulés de cet automate sont les suivantes :

- Le taux de C02 (PPM)
- La luminance (cd/m2)
- La température (°C)

Pour que le tunnel puisse être mis en service, il doit respecter les normes de sureté en vigueur, soit un taux de C02 inférieur ou égal à 800 PPM, une luminance supérieure ou égale à 300 cd/m2 et une température inférieure ou égale à 25 C°, et ce, dans les sections NORD et SUD du tunnel.

Alors que tout semblait normal la veille lors des vérifications, les équipes de conduite du tunnel découvrent avec stupéfaction, le matin même l'inauguration, que les ventilateurs et les spots sont éteints et que le poste Enedis de Grenelle alimentant le tunnel est hors service. Ils se sont alors empressés d'allumer le groupe électrogène de secours puis de redémarrer le process de régulation du tunnel via l'interface SCADA. Cependant, au moment de se connecter sur l'interface via le compte ayant les privilèges pour envoyer des consignes, ils se sont aperçus que le compte avait été supprimé pendant la nuit et qu'ils n'ont à présent accès qu'au compte en lecture seule.

Sans solutions, les épuipes de conduite font appel à vos compétences pour remettre le tunnel en service avant l'arrivée des VIP. A votre arrivée tardive, il ne vous reste qu'une minute pour remédier à la situation. Il n'y a pas une minute à perdre...

Dans cette première partie de l'épreuve, vous devez retrouvez un token caché dans les données d'identification de l'automate pour vous permettre d'accèder à la deuxième partie de l'épreuve.

Dans cette deuxième partie de l'épreuve, vous devez envoyer des consignes à l'automate afin d'obtenir des valeurs de process répondant aux normes de sureté (déjà fournies au dessus), le tout en moins d'une minute. Vous avez également accès à l'interface SCADA en lecture seule via le token récupéré dans la première partie de l'épreuve.

Ces serveurs sont accessibles à ces adresses :

- automate : `nc tunnel-routier.fcsc.fr 502`
- interface web SCADA du tunnel : https://tunnel-routier.fcsc.fr/.

Important. Chaque nouvelle connexion TCP à l'automate sur le port 502 implique la réinitialisation de l'automate, ainsi, un nouveau token est généré à chaque nouvelle connexion. Pour compléter les deux étapes de ce challenge dans le temps imparti, vous ne devrez donc pas couper votre connexion TCP et effectuer toutes vos requêtes dans la même connexion TCP.

<div>
  <img src="/posts/fcsc2026_tunnel_routier_1-2/images/tunnel.png" alt="tunnel">
  </br>
</div>

### Interface SCADA

Voici l'interface SCADA à l'adresse https://tunnel-routier.fcsc.fr/ mentionnée dans le document.

<div>
  <img src="/posts/fcsc2026_tunnel_routier_1-2/images/scada-1.png" alt="tunnel">
  </br>
</div>

---

## Résolution

Dans cette première partie, on cherche simplement à récupérer le token d'accès à l'automate SCADA, ce token serait caché dans les **données d'identification** de ce dernier.

### Protocole de communication

Avant toute chose, il faut déterminer quel protocole on va devoir utiliser pour communiquer avec le SCADA, entre le Modbus TCP, l'OPC-UA et les protocoles propriétaires, il y a beaucoup de possibilitées.

D'après la documentation, on sait que la communication entre le PLC et le SCADA se fait via le port TCP/502, en regardant dans le registre [IANA](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) (registre gérant, entre autre, les assignation de ports) et en cherchant le port **502**, on peut voir qu'il est réservé à **Modbus**, c'est donc ce que l'on va utiliser.

### Registre

Il faut maintenant identifier le registre à lire. En regardant la [documentation](https://www.modbus.org/file/secure/modbusprotocolspecification.pdf) de Modbus, on peut voir la fonction **43 / 14 (0x2B / 0x0E) Read Device Identification** qui est exactement ce que l'on veut.

### Code Python

Avec toutes ces informations, on peut écrire notre code Python en utilisant la librairie [PyModbus](https://pymodbus.readthedocs.io/en/latest/).

En cherchant **Read Device Identification** dans la [documentation](https://pymodbus.readthedocs.io/en/latest/) de PyModbus on trouve la fonction [read_device_information](https://pymodbus.readthedocs.io/en/latest/source/client.html#pymodbus.client.mixin.ModbusClientMixin.read_device_information).

{{< admonition type=note title="Note" >}}
Le fait d'utiliser `result.information` pour afficher le token est une information assez difficile à trouver, on peut voir que `read_device_information` retourne le dictionnaire **information** [ici](https://pymodbus.readthedocs.io/en/latest/source/library/pymodbus.html#pymodbus.pdu.mei_message.ReadDeviceInformationResponse)
{{< /admonition >}}

```py
from pymodbus.client import ModbusTcpClient

client = ModbusTcpClient('tunnel-routier.fcsc.fr')
client.connect()
result = client.read_device_information()
print(result.information)
```

On a en retour notre token: 

`{0: b'Siemens AG ', 1: b'S7-300 315-2EH13-0AB0 (Your token is : d3699c7e454f5fb5deeb7c6c637e44c5)', 2: b'3.0.33'}`

### Flag

On a plus qu'a le rentrer dans l'[interface SCADA](https://tunnel-routier.fcsc.fr/) et voici notre flag.

<div>
  <img src="/posts/fcsc2026_tunnel_routier_1-2/images/scada-2.png" alt="tunnel">
</div>

---
## Sources
- https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
- https://pymodbus.readthedocs.io/en/latest/
- https://pymodbus.readthedocs.io/en/latest/source/library/pymodbus.html#pymodbus.pdu.mei_message.ReadDeviceInformationResponse
- https://www.modbus.org/docs/Modbus_Application_Protocol_V1_1b3.pdf
