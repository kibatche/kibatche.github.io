+++
title = "Explorer la spécification HTML : Foreign Content et autres joyeusetés (2/4)"
date = "2026-05-02"
template = "page.html"
+++
## Introduction

Dans le premier article de cette série, nous avons vu les bases de la spécification HTML en abordant certaines variables essentielles à la construction d'un arbre et sa réparation, ainsi que la méthode de reconstruction de l'arbre.

Le but est d'aborder la spécification par sa subversion. Or, en terme de subversion, le plus classique concernant la construction d'un arbre est la mutation, qui donne notamment lieu à une vulnérabilité nommée *mutation XSS*.

Comme nous l'avons précédemment évoqué, le DOM est **dynamique** : chaque token émis est immédiatement consommé par l'étape de construction, et cette opération peut changer la structure même de ce qui est en train d'être construit, par le biais de plusieurs opérations, par exemple l'*AFE* que nous avons étudié (`Formatage des Éléments Actifs`).

Les briques de l'arbre sont les *nodes*  - nœuds -, des abstractions programmatiques qui embarquent au sein de leur structure de nombreuses variables permettant de les traiter d'une certaine façon à un instant T.

Dans la spécification HTML il existe de nombreux moyens de muter un arbre. Mais l'un des plus connus est sans aucun doute le jeu avec les *namespaces* - espaces de nom -, une des variables présentes dans un nœud sous la dénomination de *namespaceURI*.

Dans cet article, nous allons donc étudier un peu ces *namespaces* et détailler l'étape de parsing qui traite des *namespaces* : le mode `in foreign content`. Par la suite, nous allons détailler quelques autres opérations présentes au sein de la spécification qui conduisent également à des mutations, avant d'attaquer - enfin ! - .


![letsgo](content/explorer_la_spec_html5_foreign_content/let's_go.png)

## Les *namespaces* et les points d'intégration

Les *namespaces* permettent de classifier un élément et ainsi lui appliquer des règles spécifiques.

Pour rappel, trois *namespaces* sont intégrés dans la spécification HTML :
- Le `xhtml`, le *namespace* propre au HTML
- Le `svg`, le *namespace* du SVG
- Le `mathml`, le *namespace* du MathML

Ainsi, un élément `a` dans le *namespace* `xhtml` n'est pas le même élément qu'en `svg` ou `mathml`, car leur variable `namespaceURI` (dans l'interface `Element`) diffère :

{{domexplorer(id="eyJpbnB1dCI6IjxhPjwvYT5cbjxzdmc+XG48YT48L2E+XG48L3N2Zz5cbjxtYXRoPlxuPGE+PC9hPlxuPC9tYXRoPiIsInBpcGVsaW5lcyI6W3siaWQiOiJicGVucTdneCIsIm5hbWUiOiJEb20gVHJlZSIsInBpcGVzIjpbeyJuYW1lIjoiRG9tUGFyc2VyIiwiaWQiOiJiOTNqeXVlZyIsImhpZGUiOmZhbHNlLCJza2lwIjpmYWxzZSwib3B0cyI6eyJ0eXBlIjoidGV4dC9odG1sIiwic2VsZWN0b3IiOiJib2R5Iiwib3V0cHV0IjoiaW5uZXJIVE1MIiwiYWRkRG9jdHlwZSI6dHJ1ZX19XX1dfQ==")}}

Lorsque, dans un *namespace* donné - `mathml` ou `svg`, nous passons momentanément au *namespace* `xhtml`, on parle de *context switch*, en bon français : de changement de contexte.

Cela est utile pour intégrer du HTML au sein des *namespaces* étrangers. L'élément HTML (ou les éléments) sera traité avec les règles du HTML et pas du *namespace* ou l'élément s'insère.

Dans la spécification HTML, le mécanisme qui permet ce changement de contexte est nommé `integration point`. Ce point d'intégration a deux variantes :

1. [*MathML Text Integration Points*](https://html.spec.whatwg.org/multipage/parsing.html#mathml-text-integration-point)

Ils sont au nombre de 5. Ils sont utiles pour intégrer du HTML au sein de balises faisant parties du *namespace* `mathml` :

- `<mi>`   *identifier*
- `<mo>`   *operator*
- `<mn>`   *number*
- `<ms>`   *string literal*
- `<mtext>` *text*

Ci-dessous un exemple montrant une balise `a` intégrée en tant qu'élément HTML au sein de l'élément `ms`, et un autre non intégré au sein de cet élément :

{{domexplorer(id="eyJpbnB1dCI6IjxtYXRoPlxuPG1zPlxuPGE+eGh0bWw8L2E+XG48L21zPlxuPGE+bWF0aG1sPC9hPiIsInBpcGVsaW5lcyI6W3siaWQiOiJicGVucTdneCIsIm5hbWUiOiJEb20gVHJlZSIsInBpcGVzIjpbeyJuYW1lIjoiRG9tUGFyc2VyIiwiaWQiOiJiOTNqeXVlZyIsImhpZGUiOmZhbHNlLCJza2lwIjpmYWxzZSwib3B0cyI6eyJ0eXBlIjoidGV4dC9odG1sIiwic2VsZWN0b3IiOiJib2R5Iiwib3V0cHV0IjoiaW5uZXJIVE1MIiwiYWRkRG9jdHlwZSI6dHJ1ZX19XX1dfQ==")}}

2. [*HTML integration point*](https://html.spec.whatwg.org/multipage/parsing.html#html-integration-point)

Ils sont également au nombre de 5, mais s'applique, selon l'élément, à un *namespace* spécifique :
- `<annotation-xml encoding="text/html">` -> `mathml`
- `<annotation-xml encoding="application/xhtml+xml">` -> `mathml`
- `<foreignObject>` -> `svg`
- `<desc>` -> `svg`
- `<title>` -> `svg`

Voici par exemple l'élément `desc` contenant une balise `a` dans le contexte `svg`, puis `mathml`. On remarque que dans le contexte `mathml`, l'élément `a` possède le *namespace* `mathml` et non pas `xhtml` :

{{domexplorer(id="eyJpbnB1dCI6Ijxzdmc+XG4gIDxkZXNjPlxuICAgIDxhPjwvYT5cbiAgPC9kZXNjPlxuPC9zdmc+XG48bWF0aD5cbiAgPGRlc2M+XG4gICAgPGE+PC9hPlxuICA8L2Rlc2M+XG48L21hdGg+IiwicGlwZWxpbmVzIjpbeyJpZCI6ImJwZW5xN2d4IiwibmFtZSI6IkRvbSBUcmVlIiwicGlwZXMiOlt7Im5hbWUiOiJEb21QYXJzZXIiLCJpZCI6ImI5M2p5dWVnIiwiaGlkZSI6ZmFsc2UsInNraXAiOmZhbHNlLCJvcHRzIjp7InR5cGUiOiJ0ZXh0L2h0bWwiLCJzZWxlY3RvciI6ImJvZHkiLCJvdXRwdXQiOiJpbm5lckhUTUwiLCJhZGREb2N0eXBlIjp0cnVlfX1dfV19")}}


Nous avons (presque ;-) ) tous les éléments en place pour étudier le mode de *parsing* `in foreign content` !

## Le mode `in foreign content`

Lorsque l'agent utilisateur doit parser du `svg` ou du `mathml`, il doit suivre les règles du mode `in foreign content`.

Ces règles sont pour ainsi dire, avec quelques autres, les bases des XSS par mutation. En effet, l'idée est de revenir, d'une façon ou d'une autre, dans un contexte `xhtml`, mais de le faire en exploitant des ambiguïtés que les développeurs d'un agent utilisateur quelconque peuvent louper ou encore comprendre différemment de l'intention des spécificateurs.

Que nous dit en substance cet ensemble de règles ?

Voyons la première chose importante :

> A start tag whose tag name is one of: "b", "big", "blockquote", "body", "br", "center", "code", "dd", "div", "dl", "dt", "em", "embed", "h1", "h2", "h3", "h4", "h5", "h6", "head", "hr", "i", "img", "li", "listing", "menu", "meta", "nobr", "ol", "p", "pre", "ruby", "s", "small", "span", "strong", "strike", "sub", "sup", "table", "tt", "u", "ul", "var"
> A start tag whose tag name is "font", if the token has any attributes named "color", "face", or "size"
> An end tag whose tag name is "br", "p"
> [Parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors).
>
>While the [current node](https://html.spec.whatwg.org/multipage/parsing.html#current-node) is not a [MathML text integration point](https://html.spec.whatwg.org/multipage/parsing.html#mathml-text-integration-point), an [HTML integration point](https://html.spec.whatwg.org/multipage/parsing.html#html-integration-point), or an element in the [HTML namespace](https://infra.spec.whatwg.org/#html-namespace), pop elements from the [stack of open elements](https://html.spec.whatwg.org/multipage/parsing.html#stack-of-open-elements).
>
>Reprocess the token according to the rules given in the section corresponding to the current [insertion mode](https://html.spec.whatwg.org/multipage/parsing.html#insertion-mode) in HTML content.

Ce que nous spécifie cette portion est que tout un ensemble d'éléments ouvrants (`b` ou encore `font` avec un attribut `color`, `face` ou `size` etc.) et deux éléments fermants, en l'espèce `br` et `p`, provoquent **un `pop` des éléments de la pile des éléments ouverts** **si et seulement si** le nœud courant **n'est pas** un **point d'intégration texte MathML**, n'est pas **un point d'intégration HTML** et si le nœud n'est pas du HTML (entendu comme ayant l'attribut *namespaceURI* égal à `xhtml`).

Concrètement, quelles sont les conséquences ?

Exemple :

{{domexplorer(id="eyJpbnB1dCI6Ijxzdmc+XG48YT5EYW5zIGxlIG5vZXVkIFNWRzwvYT5cbjxiPkhvcnMgZHUgc3ZnPC9iPlxuPC9zdmc+IiwicGlwZWxpbmVzIjpbeyJpZCI6InJ0N3ZqdjExIiwibmFtZSI6IkRvbSBUcmVlIiwicGlwZXMiOlt7Im5hbWUiOiJEb21QYXJzZXIiLCJpZCI6Imo1d2J6dGY4IiwiaGlkZSI6ZmFsc2UsInNraXAiOmZhbHNlLCJvcHRzIjp7InR5cGUiOiJ0ZXh0L2h0bWwiLCJzZWxlY3RvciI6ImJvZHkiLCJvdXRwdXQiOiJpbm5lckhUTUwiLCJhZGREb2N0eXBlIjp0cnVlfX1dfV19")}}

On remarque très clairement que l'élément `b` est **sorti** du nœud `svg` . En effet, `b` fait partie de la liste des éléments qui provoquent un `pop`. Et il en va ainsi de l'ensemble des éléments cités dans les règles ci-dessus.

Par exemple, de façon surprenante de prime abord, avec un élément fermant `p` :

{{domexplorer(id="eyJpbnB1dCI6Ijxzdmc+XG48YT5EYW5zIGxlIG5vZXVkIFNWRzwvYT5cbjwvcD5cbjxhPkRlaG9ycyAhPC9hPlxuPC9zdmc+IiwicGlwZWxpbmVzIjpbeyJpZCI6InJ0N3ZqdjExIiwibmFtZSI6IkRvbSBUcmVlIiwicGlwZXMiOlt7Im5hbWUiOiJEb21QYXJzZXIiLCJpZCI6Imo1d2J6dGY4IiwiaGlkZSI6ZmFsc2UsInNraXAiOmZhbHNlLCJvcHRzIjp7InR5cGUiOiJ0ZXh0L2h0bWwiLCJzZWxlY3RvciI6ImJvZHkiLCJvdXRwdXQiOiJpbm5lckhUTUwiLCJhZGREb2N0eXBlIjp0cnVlfX1dfV19")}}

Cette règle est essentielle quelque part : si un parseur ne suivait pas cette règle, cela signifierait qu'un élément non autorisé resterait dans le *namespace* `svg`.

Par exemple, avec DomPurify 2.0.0, ainsi que c'est expliqué dans [cet article](https://sechub.in/view/1851825), il était possible d'exploiter le *payload* suivant avec une ancienne version de chrome (l'exemple est reconstruit) :

![](content/explorer_la_spec_html5_foreign_content/chrom_mxss_exploit.png)

Le souci principal était que le chrome d'alors ne prenait pas en compte `</p>` comme briseur de contenu.

En résultait une normalisation[^1], transformant `</p>` en `<p></p>` qui lui -  au second parsing - brisait bien bien le nœud.

![perfermanman](content/explorer_la_spec_html5_foreign_content/perfermaman.png)

Dans tout ce processus, intervient la balise `style` qui a une signification différente dans un contexte HTML : elle fait passer la machine à état en `RAWTEXT`, [comme cela est détaillé dans cette portion de la spécification](https://html.spec.whatwg.org/multipage/parsing.html#parsing-html-fragments:rawtext-state).

Ci-dessous, on voit bien le comportement :

{{domexplorer(id="eyJpbnB1dCI6Ijxzdmc+PHN0eWxlPjxhPjwvYT48L3N0eWxlPjwvc3ZnPlxuPHN0eWxlPjxhPjwvYT5cbiIsInBpcGVsaW5lcyI6W3siaWQiOiI4dHgxemY2aCIsIm5hbWUiOiJEb20gVHJlZSIsInBpcGVzIjpbeyJuYW1lIjoiRG9tUGFyc2VyIiwiaWQiOiJtNHI1ZnBtcSIsImhpZGUiOmZhbHNlLCJza2lwIjpmYWxzZSwib3B0cyI6eyJ0eXBlIjoidGV4dC9odG1sIiwic2VsZWN0b3IiOiJib2R5Iiwib3V0cHV0IjoiaW5uZXJIVE1MIiwiYWRkRG9jdHlwZSI6dHJ1ZX19XX1dfQ==")}}

La balise `a` est vue comme un élément en contexte `svg` dans une balise `style` et comme du texte, en contexte `HTML` dans une balise `style`.

Penchons nous rapidement sur cet état `RAWTEXT`.
## L'état `RAWTEXT`

Cet état est particulièrement apprécié. La spécification impose que, dès qu'un élément déclenche un tel état, les tokens soient consommés jusqu'à trouver une balise fermante qui corresponde à la balise ouvrante, dans l'exemple ci-dessus, `<style>`.

Et ceci, **n'importe où au sein du document tant que la condition n'est pas satisfaite**.

Ce qui veut dire que, dans un contexte `HTML`, mettre `</style>` dans un attribut va casser cet attribut !

{{domexplorer(id="eyJpbnB1dCI6Ijxzdmc+PHN0eWxlPjxhIGlkPVwiPC9zdHlsZT5cIj48L3N2Zz5cbjxzdHlsZT48YSBpZD1cIjwvc3R5bGU+PFBBWUxPQUQ+XCI8L2E+XG4iLCJwaXBlbGluZXMiOlt7ImlkIjoiOHR4MXpmNmgiLCJuYW1lIjoiRG9tIFRyZWUiLCJwaXBlcyI6W3sibmFtZSI6IkRvbVBhcnNlciIsImlkIjoibTRyNWZwbXEiLCJoaWRlIjpmYWxzZSwic2tpcCI6ZmFsc2UsIm9wdHMiOnsidHlwZSI6InRleHQvaHRtbCIsInNlbGVjdG9yIjoiYm9keSIsIm91dHB1dCI6ImlubmVySFRNTCIsImFkZERvY3R5cGUiOnRydWV9fV19XX0=")}}

Et c'est la clé de la mutation exposée dans la section précédente : l'élement `style`, au départ inoffensif dans un contexte `svg`, est devenu dévastateur une fois ce contexte brisé au second parsing.

L'élément `style` n'est cependant pas le seul à déclencher cet état. Il y a également :
- [xmp](https://html.spec.whatwg.org/multipage/obsolete.html#xmp)
- [iframe](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#the-iframe-element)
- [noembed](https://html.spec.whatwg.org/multipage/obsolete.html#noembed)
- [noframes](https://html.spec.whatwg.org/multipage/obsolete.html#noframes)

L'élément [noscript](https://html.spec.whatwg.org/multipage/scripting.html#the-noscript-element) quant à lui est particulier : il déclenche cet état dès lors que le mode `scripting` n'est pas `Disable` ([voir ici](https://html.spec.whatwg.org/multipage/parsing.html#scripting-mode)).

## Quelques autres opérations intéressantes au sein de la spécification

En somme, l'idée est de trouver des comportements au sein de la spécification qui sont sujets à erreur. Nous avons vu ici le mode `in foreign content` et létat, mais il y a de nombreuses, très nombreuses autres règles du HTML qui amènent à de la confusion.

Nous allons maintenant voir quelques autres aspects notables.

### L'état `RCDATA`

Il agit [exactement](https://html.spec.whatwg.org/#rcdata-state) comme l'état `RAWTEXT` à l'exception que le mode `RCDATA` décode  les entités HTML :

{{domexplorer(id="eyJpbnB1dCI6Ijx0ZXh0YXJlYT4mbHQ7YSZndDs8L3RleHRhcmVhPlxuPHhtcD4mbHQ7YSZndDs8L3htcD4iLCJwaXBlbGluZXMiOlt7ImlkIjoicnQ3dmp2MTEiLCJuYW1lIjoiRG9tIFRyZWUiLCJwaXBlcyI6W3sibmFtZSI6IkRvbVBhcnNlciIsImlkIjoiajV3Ynp0ZjgiLCJoaWRlIjpmYWxzZSwic2tpcCI6ZmFsc2UsIm9wdHMiOnsidHlwZSI6InRleHQvaHRtbCIsInNlbGVjdG9yIjoiYm9keSIsIm91dHB1dCI6ImlubmVySFRNTCIsImFkZERvY3R5cGUiOnRydWV9fV19XX0=")}}

Les éléments `RCDATA` sont : [`title`](https://html.spec.whatwg.org/#the-title-element) et [`textarea`](https://html.spec.whatwg.org/#the-textarea-element).

### L'état `CDATA`

[Cet état](https://html.spec.whatwg.org/#cdata-section-state) ne fonctionne que dans les espaces de noms `svg` et `mathml`. Dans ces espaces de noms, il agit comme `RAWTEXT`, sinon [le parseur passe](https://html.spec.whatwg.org/#markup-declaration-open-state) dans l'état [`bogus comment`](https://html.spec.whatwg.org/#bogus-comment-state) après avoir inséré le commentaire `[CDATA[`. Cet état consomme tous les caractères jusqu'à tomber sur le caractère `>`.

{{domexplorer(id="eyJpbnB1dCI6IjxhPjwhW0NEQVRBWzxhPl1dPlxuPHN2Zz48IVtDREFUQVs8YT5dXT4iLCJwaXBlbGluZXMiOlt7ImlkIjoicnQ3dmp2MTEiLCJuYW1lIjoiRG9tIFRyZWUiLCJwaXBlcyI6W3sibmFtZSI6IkRvbVBhcnNlciIsImlkIjoiajV3Ynp0ZjgiLCJoaWRlIjpmYWxzZSwic2tpcCI6ZmFsc2UsIm9wdHMiOnsidHlwZSI6InRleHQvaHRtbCIsInNlbGVjdG9yIjoiYm9keSIsIm91dHB1dCI6ImlubmVySFRNTCIsImFkZERvY3R5cGUiOnRydWV9fV19XX0=")}}

### Le foster parenting

Le [`foster parenting`](https://html.spec.whatwg.org/#foster-parent) est un mécanisme qui concerne les éléments  [table](https://html.spec.whatwg.org/#the-table-element), [tbody](https://html.spec.whatwg.org/#the-tbody-element), [tfoot](https://html.spec.whatwg.org/#the-tfoot-element), [thead](https://html.spec.whatwg.org/#the-thead-element) et [tr](https://html.spec.whatwg.org/#the-tr-element). Au sein de ces éléments, ne sont autorisés que les éléments suivants : `<caption>`, `<colgroup>`, `<col>`, `<tbody>`, `<tfoot>`, `<thead>,` `<tr>`, `<td>`, `<th>`, `<script>`, `<template>`, `<style>`. Si un élément non autorisé est présent, il est inséré avant l'élément concerné par le `foster parenting`.

{{domexplorer(id="eyJpbnB1dCI6Ijx0YWJsZT48c3R5bGU+PC9zdHlsZT48L3RhYmxlPlxuPCEtLS0tPlxuPHRhYmxlPjxhPjwvYT48L3RhYmxlPiIsInBpcGVsaW5lcyI6W3siaWQiOiJydDd2anYxMSIsIm5hbWUiOiJEb20gVHJlZSIsInBpcGVzIjpbeyJuYW1lIjoiRG9tUGFyc2VyIiwiaWQiOiJqNXdienRmOCIsImhpZGUiOmZhbHNlLCJza2lwIjpmYWxzZSwib3B0cyI6eyJ0eXBlIjoidGV4dC9odG1sIiwic2VsZWN0b3IiOiJib2R5Iiwib3V0cHV0IjoiaW5uZXJIVE1MIiwiYWRkRG9jdHlwZSI6dHJ1ZX19XX1dfQ==")}}

### Toujours plus !

Si vous en souhaitez toujours plus, vous pouvez consulter ces liens :

- https://sonarsource.github.io/mxss-cheatsheet/ : Plusieurs techniques de mutation XSS présentées. Par d'explications techniques sur les pourquoi du comment, mais ça reste cool.
- https://mizu.re/post/exploring-the-dompurify-library-bypasses-and-fixes && https://mizu.re/post/exploring-the-dompurify-library-hunting-for-misconfigurations : Les articles de Mizu dont nous avons parlé dans le premier article.
- Mario Heiderich — "mXSS Attacks: Attacking well-secured Web-Applications by using innerHTML Mutations"
- kindone09 — ["Beyond `<script>`: Weaponizing `<?` and `<![` for Next-Gen XSS"](https://kindone09.medium.com/beyond-script-weaponizing-and-for-next-gen-xss-792716317def)
- ryotak — ["Bypassing DOMPurify with good old XML"](https://flatt.tech/research/posts/bypassing-dompurify-with-good-old-xml/)
- Jorian Woltjer — [Mutation XSS](https://jorianwoltjer.com/blog/p/research/mutation-xss)
- Cure53 — https://cure53.de/


Spécs :

- [WHATWG — Parsing Foreign Content](https://html.spec.whatwg.org/multipage/parsing.html#parsing-main-inforeign)
- [WHATWG — MathML Text Integration Point](https://html.spec.whatwg.org/multipage/parsing.html#mathml-text-integration-point)
- [WHATWG — HTML Integration Point](https://html.spec.whatwg.org/multipage/parsing.html#html-integration-point)
- [WHATWG — Template Element](https://html.spec.whatwg.org/multipage/scripting.html#the-template-element)
- [MathML Core Spec](https://w3c.github.io/mathml-core/)
- [SVG 2 Spec](https://svgwg.org/svg2-draft/)
- [W3C MathML3 — xmlns note](https://www.w3.org/TR/MathML3/chapter2.html)
- https://wpt.fyi/results/domparsing?label=master&label=experimental&aligned : ;) Check this out, c'est intéressant comme site. :-)

Et de nombreux autres !
## Conclusion

Nous avons vu plusieurs concepts qui nous permettent de potentiellement muter un arbre.

Dans la suite de ces articles, nous allons nous intéresser aux outils que nous allons utiliser pour analyser le parseur lexbor.

Enfin, le dernier article abordera ces analyses ainsi que le désinfecteur HTML de `Symfony` !

[^1]: Par le biais du DomParser de Chrome
