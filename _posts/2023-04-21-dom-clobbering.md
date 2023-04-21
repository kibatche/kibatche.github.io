---
title: DOM Clobbering, une vulnérabilité bizarre
categories:
- Vulnérabilités Web
- web
tags:
- javascript
- vulnérabilités
- web
---

## Introduction

C'est à l'occasion de la lecture d'un write-up[^footnote]  d'un camarade de l'associaton APT42 que j'ai entendu parler la première fois de cette vulnérabilité. Je dois bien avouer que je n'avais absolument rien compris à ce moment, et c'est lors de la résolution d'un challenge sur root-me que j'ai pu vraiment saisir en quoi elle consiste. Ce court article vise à présenter ce que j'ai acquis comme compréhension, et les façons d'éviter de la mitiger.
	
### Le DOM Clobbe-quoi ?

Que peut-on attendre d'un langage écrit en 10 jours, omniprésent sur les internets, et tributaire de plus 30 ans d'itérations ? Eh bien des trucs chelous, qui me font parfois penser au javascript comme à une sorte d'énorme passoire que les développeuses et développeurs se doivent de patcher afin d'éviter de très sérieux problèmes.

Parmi les vulnérabilités côté client, celle qui est la plus connue est très certainement l'attaque  XSS. Selon sa nature, elle peut être mitigée via différentes stratégies de sécurité, comme les CSP. Une vulnérabilité comme le DOM Clobbering, qui me semble relativement peu connue, n'est pas mitigée par les CSP ou autres stratégies, et concerne directement le code en lui-même.

Pour synthétiser, le DOM-Clobbering vise à polluer une variable globale par une variable qu'un attaquant peut manipuler. La finalité sera de modifier le comportement de l'application - et le DOM -, afin de potentiellement mener des attaques de type XSS ou autre.

Et ceci est possible même avec une politique CSP très restrictive ou des désinfectants (sanitizers) comme DOM Purify ! En effet, il suffit pour l'attaquant de pouvoir manipuler une page html (y ajouter des éléments par exemple), et on verra qu'une simple ancre `<a>` suffit à fiche une bazar monstre !

Nous verrons la façon la plus commune (et simple) de déclencher cette vulnérabilité.


### Un exemple 

Prenons un exemple très simple tiré du site OWASP (et un peu changé).

```html
<--html>
<script>
    let newUrl = window.newURL.url || 'https://url.com';
    // code javascript pour changer de page, assigner une valeur à la source d'un script etc...
</script>
```

Si on est en capacité de créer un élément dans le document html, alors cela pourrait mener à une attaque de type XSS, rediriger vers une page malicieuse etc.

Par exemple, on pourrait créer l'ancre `<a>` avec pour id "newURL".

```html
<a id=newURL><a id=newURL name=url href="https://pwned.com">
```

On a même créé deux balises `<a>`. Pourquoi ? Car sur les navigateurs basés sur chromium (chrome, les derniers edge etc), le fait de créer plusieurs éléments avec le même id crée une collection HTML. 

L'ancre `<a>` avec l'id "newURL" devient donc une collection, et ce type de balise est très utile dans notre cas, car il peut définir une url via href, et est très peu suspecte (par exemple, elle n'est pas supprimée par défaut par DOMPurify).

Le fait de donner la valeur "url" à "name" permet précisément de "d'écraser" cette propriété de la variable globale, et de lui donner une autre valeur que celle attendue.

On peut ainsi multiplier les possibilités à l'infini. On pourrait accéder et donner une valeur à window.x.y, même si ni x ni y n'existent véritablement : il suffit de les créer ! Et s'ils existent, il suffit de les changer.

Il est à noter que ici nous n'avons vu qu'une possibilité de clobbering. Vous pouvez voir d'autres exemples sur le site [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/DOM_Clobbering_Prevention_Cheat_Sheet.html), le site [domclob.xyz](https://domclob.xyz/) (qui présente de très nombreux payloads), et sur [portswigger](https://portswigger.net/web-security/dom-based/dom-clobbering).

Si vous souhaitez mettre en pratique votre compréhension de cette vulnérabilité, root-me propose un [challenge](https://www.root-me.org/fr/Challenges/Web-Client/DOM-Clobbering) assez ardu, ou encore deux labs sur PortSwigger, pas du tout simples non plus.

Terjank a écrit un [write-up](https://terjanq.medium.com/dom-clobbering-techniques-8443547ebe94) pour un challenge twitter, et donne de très bonnes explications sur la marche à suivre pour "clobberer" une variable.

Avec toutes ces ressources, vous devriez être en mesure de comprendre cette vulnérabilité étrange !


### Comment mitiger ce type de vulnérabilité ?

Nous avons vu que le fait d'écraser une variable globale par une créée par un attaquant ne met pas en jeu des choses aussi communément évitables qu'une balise script.

Afin de se protéger, il ne semble pas exister 36 solutions.

Il faut éviter d'assigner la valeur d'un objet global en utilisant l'opérateur logique `||`.

Ensuite, même si ça peut paraître un peu bête, il faut toujours vérifier qu'un objet est bien du type de celui attendu. Pour terminer, il peut être intéressant d'éviter au maximum l'emploi de variables globales, qui sont les vecteurs de cette vulnérabilité.

[^footnote]: [Ecrit](https://writeup.owalid.com/writeups/batchcraft-potions) par owalid, un excellent article pour un challenge du HTB University CTF 2022