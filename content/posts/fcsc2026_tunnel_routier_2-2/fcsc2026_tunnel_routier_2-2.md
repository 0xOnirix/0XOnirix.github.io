---
title: "FCSC 2026 - Tunnel Routier 2/2"
date: 2026-04-12T12:00:00+00:00
draft: false
toc: true
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
  <img src="/posts/fcsc2026_tunnel_routier_2-2/images/logo_fcsc.svg" alt="Logo_FCSC" style="width: 50%">
  </br>
</div>

Ceci est le write-up de la deuxième partie du challenge [Misc](/tags/misc/) **Tunnel Routier** du CTF **France CyberSecurity Challenge** ([FCSC](https://fcsc.fr/)) édition 2026 organisé par l'ANSSI.

## Énoncé

>Dans cette deuxième partie de l'épreuve, vous devez envoyer des consignes à l'automate afin d'obtenir des valeurs de process répondant aux normes de sureté (fournie dans la description détaillée), le tout en moins d'une minute. Vous avez également accès à l'interface SCADA en lecture seule via le token récupéré dans la première partie de l'épreuve.
>
>Note: La description du système est disponible sur une [page dédiée](https://fcsc.fr/tunnel-routier-76bea48861178b0c).
>
>`nc tunnel-routier.fcsc.fr 502` 

### Documentation

Voici la documentation présente à l'adresse: https://fcsc.fr/tunnel-routier-76bea48861178b0c


**Tunnel routier : documentation**

Le tunnel de Grenelle d'une longueur de 5km doit être inauguré ce jour, pour cela des responsables locaux ont fait le déplacement pour assister à sa première mise en service. Le système de contrôle-commande de ce tunnel est assuré par un automate programmable industriel (PLC) communiquant avec le SCADA au travers d'un protocole industriel sur le port TCP/502. Ce système permet de réguler des valeurs de process offrant une traversée du tunnel en toute sécurité pour les usagers. Les valeurs de process régulés de cet automate sont les suivantes :

- Le taux de C02 (PPM)
- La luminance (cd/m²)
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
  <img src="/posts/fcsc2026_tunnel_routier_2-2/images/tunnel.png" alt="tunnel">
  </br>
</div>

---

## Identification

Dans cette 2ème partie, on doit remettre en fonctionnement le tunnel en moins de **1 minute**.

Pour ça, on doit agir sur 3 grandeurs physiques:

- Le taux de C02 (PPM)
- La luminance (cd/m2)
- La température (°C)

Pour commencer, on doit identifier deux choses:

- Le registre des capteurs
- Le registre des actionneurs

### Identification des types de registres

Le tableau de la section **4.3 MODBUS Data model** de la [documentation](https://www.modbus.org/docs/Modbus_Application_Protocol_V1_1b3.pdf) MODBUS nous donne le détail des différents registres:

<div>
  <img src="/posts/fcsc2026_tunnel_routier_2-2/images/register_table.png" alt="tunnel">
  </br>
</div>

Commençont par le registre des **capteurs**, on sait que:

- Ce sont des *Input*
- Très probablement en lecture seule
- Que ce sont des valeurs sur plus d'un bit

D'après le tableau, tout porte à croire qu'il s'agit du **Input Registers**.

Pour celui des **actionneurs**, on sait que:

- Ils doivent être modifiable (Read-Write)
- Ils doivent être sur plus d'un bit (car pas simplement on/off)

D'après le tableau, tout porte à croire qu'il s'agit du **Holding Registers**.

Chaque registre (capteur ou actionneur) peut avoir 65536 adresses, chaque adresse contient **16 bits** de données (voir le tableau des registres ci-dessus).

Si on regarde le schéma dans l'interface SCADA, on voit qu'il y a **3 capteurs** et **8 actionneurs** (4 lumières et 4 ventilateurs) par tunnel (Nord et Sud), soit un total de **6 capteurs** et **16 actionneurs**.

### Identification des adresses du registre des capteurs

Notre but ici est de savoir à qu'elle adresse est stockée qu'elle valeur de quel capteur, on commence par lire le registre **Input Registers**:

{{< admonition type=note title="Note" >}}
Pour `client.read_input_registers()`, le paramètre `address` indique l'adresse de départ et `count` le nombre de valeur à afficher, sachant que l'on a 6 capteurs en tout on met `count=6`
{{< /admonition >}}

```py
from pymodbus.client import ModbusTcpClient

# Initialise la connexion Modbus
client = ModbusTcpClient('tunnel-routier.fcsc.fr')
client.connect()

# Lit les 6 valeurs du input registers à l'adresse 0
result = client.read_input_registers(address=0, count=6)
print(result)
```

Ce qui nous donne: `ReadInputRegistersResponse(dev_id=1, transaction_id=1, address=0, count=0, bits=[], registers=[1600, 1600, 0, 0, 32, 32], status=1, retries=0)`

Or, on veut juste la partie `registers`, on ajuste alors le script en modifiant `print(result)` en `print(result.registers)`, ce qui nous donne:

`[1600, 1600, 0, 0, 32, 32]`

D'après l'ordre de grandeur, on peut en déduire que les deux premiers sont les capteurs de CO2, les deux suivants de luminance et enfin les deux derniers de température.
Il est aussi très probable que le première de la paire concerne le tunnel nord et le second le tunnel sud.

Pour résumer:

| **Adresse** | **Capteur** | **Tunnel** |
|:-----------:|:-----------:|:----------:|
|      0      |     CO2     |    Nord    |
|      1      |     CO2     |     Sud    |
|      2      |  Luminance  |    Nord    |
|      3      |  Luminance  |     Sud    |
|      4      | Température |    Nord    |
|      5      | Température |     Sud    |

{{< admonition type=note title="Note" >}}
On pourra vérifier ce tableau lors de l'identification des actionneurs
{{< /admonition >}}

### Identification des adresses du registres des actionneurs

Ensuite, on s'occupe du registre **Holding Registers**. Cette fois je vais écrire une valeur à une adresse et regarder sur quel actionneur ça agit sur le tableau de bord:

{{< admonition type=note title="Note" >}}
Ici on ne précise pas *holding registers* car c'est le seul registre écrivable (oui c'est moche mais ça existe), sinon c'est un *coil*.
{{< /admonition >}}

```py
from pymodbus.client import ModbusTcpClient

# Initialise la connexion Modbus
client = ModbusTcpClient('tunnel-routier.fcsc.fr')
client.connect()

# Affiche le token
print(print(client.read_device_information().information))

# Ecrit la valeur 200 à l'adresse 0
client.write_register(address=0, value=200)
```

Quand on regarde le tableau de bord, on peut voir que le premier ventilateur du tunnel nord s'est allumé avec pour valeur **200** (surement en tours/min):

<div>
  <img src="/posts/fcsc2026_tunnel_routier_2-2/images/dashboard-1.png" alt="tunnel">
  </br>
</div>

On continue à tester les adresses jusqu'a les avoir toutes mappées, ce qui donne:

| **Adresse** | **Actionneur** | **Numéro** | **Tunnel** |
|:-----------:|:--------------:|:----------:|:----------:|
|      0      |   Ventilateur  |      1     |    Nord    |
|      1      |   Ventilateur  |      2     |    Nord    |
|      2      |   Ventilateur  |      3     |    Nord    |
|      3      |   Ventilateur  |      4     |    Nord    |
|      4      |   Ventilateur  |      1     |     Sud    |
|      5      |   Ventilateur  |      2     |     Sud    |
|      6      |   Ventilateur  |      3     |     Sud    |
|      7      |   Ventilateur  |      4     |     Sud    |
|      8      |      Spots     |      1     |    Nord    |
|      9      |      Spots     |      2     |    Nord    |
|      10     |      Spots     |      3     |    Nord    |
|      11     |      Spots     |      4     |    Nord    |
|      12     |      Spots     |      1     |     Sud    |
|      13     |      Spots     |      2     |     Sud    |
|      14     |      Spots     |      3     |     Sud    |
|      15     |      Spots     |      4     |     Sud    |

---

## Résolution

Maintenant que l'on a tous les éléments, on va pouvoir écrire le script finale pour résoudre ce challenge.

{{< admonition type=note title="Note" >}}
Pour rappel, on doit avoir:
- CO2 <= 800
- Luminance >= 300
- Température <= 25

A noter que l'on a que 2 actionneurs pour 3 capteurs, les ventilateurs vont donc agir sur le taux de CO2 **ET** la température.
{{< /admonition >}}

```py
from pymodbus.client import ModbusTcpClient
import time
import re

def get_token(client):
    """
    Retrieve the token from the device information.

    Parameters:
    client (ModbusTcpClient): Instance of ModbusTcpClient for TCP communication.

    Returns: 
    str: The access token.
    """
    token = str(client.read_device_information().information[1])

    match = re.search(r"Your token is :\s([A-Za-z0-9]+)", token)
    token = match.group(1)

    return token

def start_fans(client, value):
    """
    Set the speed (RPM) for each fan of the North and South tunnel.

    Parameters:
    client (ModbusTcpClient): Instance of ModbusTcpClient for TCP communication.
    value (int): The RPM to set for the fans.
    """

    for i in range(0,8):
        client.write_register(address=i, value=value)

def turn_on_lights(client, value):
    """
    Set the luminance for each light of the North and South tunnel.

    Parameters:
    client (ModbusTcpClient): Instance of ModbusTcpClient for TCP communication.
    value (int): The luminance to set for the lights.
    """

    for i in range(8,16):
        client.write_register(address=i, value=value)

if __name__ == "__main__":

    client = ModbusTcpClient('tunnel-routier.fcsc.fr')
    client.connect()

    token = get_token(client)

    # The value of the fans has been tested manually until it works
    start_fans(client, 1400)

    # The value of the light is a simple addition (we need 300/tunnel with 4 lights)
    turn_on_lights(client, 75)

    while True:

        sensors = client.read_input_registers(address=0,count=6)

        co2_north = sensors.registers[0]
        co2_south = sensors.registers[1]

        temp_north = sensors.registers[4]
        temp_south = sensors.registers[5]

        if co2_north <= 800 and co2_south <= 800 and temp_north <= 25 and temp_south <= 25:

            print(f"Let's get that flag: https://tunnel-routier.fcsc.fr/{token}")
            break

        time.sleep(1)
```

Ce script va allumer les ventilateurs à **1400 tr/min** (j'ai trouvé la valeur "manuellement" en testant jusqu'a que ça fonctionne dans le temps impartis), les spots à **75 cd/m²** (on a besoin de 300cd/m² par tunnel et on a 4 spots par tunnel, donc 300/4) et va lire les valeurs des capteurs jusqu'a ce qu'elles soient dans les normes.

Une fois ces valeurs atteintes, il va afficher un lien avec le token pour récupérer le flag.

---
## Flag

On peut alors récupérer le flag via le lien donné par le script:

<div>
  <img src="/posts/fcsc2026_tunnel_routier_2-2/images/second_flag.png" alt="tunnel">
  </br>
</div>

On pourrait aussi récupérer automatiquement le flag via la balise `<div id="flag"></div>` dans le code HTML du tableau de bord.

---
## Sources
- https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
- https://pymodbus.readthedocs.io/en/latest/
- https://pymodbus.readthedocs.io/en/latest/source/library/pymodbus.html#pymodbus.pdu.mei_message.ReadDeviceInformationResponse
- https://www.modbus.org/docs/Modbus_Application_Protocol_V1_1b3.pdf
