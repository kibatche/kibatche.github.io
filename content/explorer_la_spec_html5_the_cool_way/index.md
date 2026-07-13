+++
title = "Explorer la spécification HTML : The cool way (1/4)"
date = "2026-03-28"
template = "page.html"
+++
## Introduction

En fin d'année 2024 et au début de l'année 2025, il y eut deux articles de Kevin Mizu sur la librairie DOMPurify : [Exploring the DOMPurify library: Bypasses and Fixes (1/2)](https://mizu.re/post/exploring-the-dompurify-library-bypasses-and-fixes) et [Exploring the DOMPurify library: Hunting for Misconfigurations (2/2)](https://mizu.re/post/exploring-the-dompurify-library-hunting-for-misconfigurations).

Ces articles explorent de façon approfondie la librairie `DOMPurify` et les moyens de la contourner à ce moment là. Pour les personnes qui ne connaissent pas ce qu'est `DOMPurify`, il s'agit d'une bibliothèque qui permet de désinfecter - `sanitize` en anglais - les entrées utilisateurs afin d'empêcher l'introduction de code malveillant comme de l'exécution de script (XSS).

Et en général, ces vulnérabilités surviennent via des mutations de l'arbre généré par le parseur HTML : le DOM. Par exemple `DOMPurify` (côté client), utilise les données générées par `DOMParseur`, la fonction générique présente dans tous les programmes prétendant implémenter la spécification `HTML`. En l'occurrence, les navigateurs, et de façon plus générique les `user-agents`, les agents utilisateurs en français.

Cette généricité permet donc à n'importe quel programme d'utiliser le standard HTML.

Et il s'avère qu'il existe pléthore d'implémentations côté serveur. 

En effet, pour différentes raisons, les langages de programmations côté serveur qui génèrent des applications web (ou des portions de) peuvent avoir besoin de représenter un DOM côté serveur, par exemple pour désinfecter une entrée utilisateur.

En `PHP` l'outil le plus connu pour générer un arbre HTML est `libxml2`. C'est un portage un peu bizarre de la spécification HTML à partir d'une librairie qui a été essentiellement créée pour parser du xml, comme son nom l'indique fort bien. Elle n'est donc absolument pas conforme à la spécification HTML telle que définie par le comité whatwg.

Depuis PHP 8.4, `libxml2` a laissé place à un autre parseur : `lexbor`, qui est exposé via l'API `DOM`. Ce parseur, créé en C par une seule personne, a pour prétention - sans que ce mot ait ici un sens péjoratif - d'implémenter la spécification HTML dans la globalité, et donc d'être *spec compliant*. Cela donne la possibilité aux programmes PHP d'utiliser une interface unifiée, débarrassée des problèmes inhérents à *libxml2*.

Sans aller plus loin pour l'instant sur ces histoires, il s'agit d'une entreprise gargantuesque : la spécification HTML porte sur plusieurs milliers de lignes, avec des formulations absconses à de nombreux endroits, des ambiguïtés sur ce qu'un programme souhaitant l'implémenter doit faire, des machines à états extrêmement difficiles à comprendre par moment, de nombreuses définitions, des cas limites aux conséquences difficilement observables par la "simple" lecture de la spécification etc.

En bref, il s'agit d'un travail d'une complexité inouïe qui occasionne forcément des bugs et des divergences avec les parseurs des navigateurs, qui eux même implémentent de façon imparfaite plusieurs points de la spécification HTML5. Et c'est précisément dans ces divergences que surgissent des mutations.

### Pourquoi une telle démarche

C'est un sujet qui pourrait paraître dépassé : quel est l'intérêt d'explorer les mécanismes de mutations sur un parseur HTML à l'heure de la toute puissance des sanitizers, du *hardening* opéré au sein des navigateurs afin de limiter au maximum les possibilités d'exploitation ? 

D'ailleurs nous, petites mains de la sécurité informatique, ne sommes-nous pas voué.es à disparaître, si crétins sommes nous face à la toute puissance de la Déesse IA, dont les Archanges Claude et GPT sont les représentants les plus féroces (et idiots diront certain.es) ?

![mots-croisés](content/explorer_la_spec_html5_the_cool_way/mot_croisés.png)

Eh oui... À quoi bon mesdames messieurs, **À QUOI BON** se péter le cerveau ? Pour la gloire ? Pour Sparte ? Pour l'argent ?

À toutes ces questions, je dirais : la volonté de savoir. Aussi bête que cela.

Ma connaissance du sujet était quasiment inexistante avant d'entamer ces quelques semaines d'apprentissage, et je souhaitais combler cette lacune.

Car malgré la complexité du sujet, il y avait un truc : ce qui était présenté par mizu était vraiment *cool* (oui juste ça c'est suffisant), mais en plus c'était intriguant. Les articles me donnaient envie d'en savoir plus. Et quoi de mieux pour étudier un sujet complexe que de le faire via les potentialités de subversion qu'il renferme ?

Ce n'est pas à proprement parler d'une méthode *Zero to Heroe* mais de mon point de vue, étudier une partie de la spécification HTML par ses abus est vraiment une façon excellente de le faire : il y a un côté amusant à la chose, un peu sorcier fou, à tester différentes charges utiles auxquelles on peut penser après la lecture de certaines parties de la spécification. Il était (et ça l'est toujours) inenvisageable pour moi de le faire de la façon conseillée par whatwg :

> This specification should be read like all other specifications. First, it should be read cover-to-cover, multiple times. Then, it should be read backwards at least once. Then it should be read by picking random sections from the contents list and following all the cross-references.

En un mot comme en cent :

![oumar_coach_bonsoir_non.png](content/explorer_la_spec_html5_the_cool_way/oumar_coach_bonsoir_non.png)

Car outre le fait que nous pouvons avoir d'autres choses à foutre que passer nos journées à lire une spécification de plusieurs milliers de pages (avec des références croisées à d'autres standard), il s'agissait aussi (surtout ?) de voir comment on pouvait *utiliser* ces abus : en effet, l'enjeu principal des mutations réside dans le contournement des mécanismes de protection, et ce n'est pas par la **seule** lecture qu'on peut tenter cela.

Cependant, cependant, cependant... C'est quand même un petit paquet de pages à lire et à comprendre. Donc soyons clair : mon propos n'est pas de rejeter la lecture en tant que telle. Mais de choisir soigneusement les portions à lire pour le but que je me suis fixé.

En effet, la spécification est connue pour contenir *certains*  passages qui peuvent occasionner des problèmes et, si d'autres peuvent éventuellement mener à certains comportements étranges, il s'agissait de *découvrir* la spécification HTML, pas de la maîtriser.

Et, afin que ce soit encore plus intéressant et motivant, de le faire avec une cible. En PHP, il n'y a pas 36 librairies de désinfection. La plus connue est sans conteste `html sanitizer` de Symfony. Ce fut donc, dans un premier temps, la matière sur laquelle j'ai travaillé.

On le verra plus tard, mais cette façon de présenter est un peu trompeuse : cet apprentissage démarré mi-février a en fait constamment dérivé vers plusieurs sujets, et ces sujets en eux-mêmes dépassent le cadre de la "simple" recherche de mutation (qui est tout sauf triviale).

Je vais donc tenter d'aborder ces différents aspects tout du long de ces X articles. Cela concernera aussi bien l'outillage que la méthodologie ou encore l'utilisation de la dite "Intelligence Artificielle" dans le cadre d'une recherche à visée pédagogique.

Le but est de présenter cette exploration, des outils, des pistes de réflexion, le tout de la façon la plus pédagogique qu'il soit afin qu'éventuellement ce puisse être utile pour d'autres personnes.

![let's go!](content/explorer_la_spec_html5_the_cool_way/let's_go.png)

# Partie I : Un peu de théorie

Si lire la spécification dans son entier n'est pas forcément une bonne idée, il n'en reste pas moins que sans idée du contenu de cette spécification, de la façon dont elle oriente les implémentations, on sera bien embêté pour comprendre quoique ce soit aux mutations. Le but de ces articles étant d'être les plus pédagogiques possible, nous allons tout d'abord voir ce que contient la fameuse spécification HTML, évidemment de façon superficielle, et quelles sont les zones d'intérêt de cette dernière.

## C'est quoi "HTML5"

Une réponse rapide serait : la suite de HTML4. Mais une réponse qui se rapprocherait plus de la vérité serait : une tentative de mettre un peu d'ordre dans le bazar incroyable que pouvait être les technologies utilisées sur le web au début des années 2000.

Plutôt que HTML5, le comité whatwg parle tout simplement de HTML. L'absence de version dénote la volonté d'unifier les règles qui régissent le langage, héritier de plusieurs technologies : le `HTML4`, le `XHTML1` - qui se base sur la syntaxe `XML` - ainsi que `DOM2` (*DOM* pour ***D**ocument **O**bject **M**odel*).

Il s'agit d'une spécification qu'on peut arbitrairement séparer en trois grandes parties :

1. **Une Partie qui concerne les différentes définitions d'éléments et concepts de HTML :**
	- [L'infrastructure commune](https://html.spec.whatwg.org/multipage/infrastructure.html#infrastructure): cette section détermine la base commune à toute la spécification (les unités informatiques utilisées, la nomenclature etc.)
	- [La sémantique, la structure ainsi que les différentes API d'un Document HTML]([Semantics, structure, and APIs of HTML documents](https://html.spec.whatwg.org/multipage/dom.html#dom): cette section détermine par exemple l'objet `Document`, ainsi que tout ce qui concerne l'implémentation des différents éléments du HTML par le biais notamment d'**interfaces**, ainsi que les attributs globaux des éléments HTML, ou encore différentes propriétés.
	- [Les éléments du HTML](https://html.spec.whatwg.org/multipage/semantics.html#semantics): c'est ce que tout le monde connaît, la définition de l'ensemble des éléments disponibles en HTML, comme la balise `a` ou encore `script`, le tout regroupé par typologie. On a ainsi des éléments typés comme des éléments de métadonnées (`meta` par exemple), des `section` etc.
	- [Les microdonnées]([Microdata](https://html.spec.whatwg.org/multipage/microdata.html#microdata): Définition d'un certains type de métadonnées utilisé au sein des documents

2. **Une partie qui concerne principalement l'implémentation programmatique de fonctionnalités HTML :**
	- [Les interactions utilisateurs](https://html.spec.whatwg.org/multipage/interaction.html#editing): Détermine certains attributs qui peuvent ou non déclencher des événements après interactions, permettent de rendre visibles ou non certains éléments auprès des utilisateurs etc. et les différentes interfaces de programmation afférentes.
	- [Le chargement des pages web](https://html.spec.whatwg.org/multipage/browsers.html#browsers): Détermine l'objet `Window`, ainsi que les interfaces`Location` ou encore `History` parmi d'autres. C'est une section qui aborde également la notion d'origine en HTML - *origin* - qui est centrale dans la sécurité informatique liée aux applications web.
	- [API des applications Web](https://html.spec.whatwg.org/multipage/webappapis.html#webappapis): Détermine de nombreuses choses comme la façon dont les scripts sont exécutés dans le contexte d'un document, l'insertion dynamique de balise, et surtout **l'interface de *parsing* du DOM ainsi que sa sérialisation via l'API `DOMParser`**.
	- [La communication](https://html.spec.whatwg.org/multipage/comms.html#comms): Détermine l'interface `MessageEvent` utilisée dans différentes façon de communiquer (au sein d'un document, entre document etc.)
	- [Les Web workers](https://html.spec.whatwg.org/multipage/workers.html#workers): Détermine les différentes interfaces liées aux workers web, qui permettent d'exécuter du javascript en tâche de fond.
	- [Le Web storage](https://html.spec.whatwg.org/multipage/webstorage.html#webstorage): Détermine les différentes interfaces liées au Web storage qui sert à stocker des données de type clé/valeur au sein du navigateur.

3. **Une partie qui concerne le parsing (donc l'analyse et le traitement syntaxique) de HTML ainsi que le rendu de HTML :**
	- [La syntaxe HTML](https://html.spec.whatwg.org/multipage/syntax.html#syntax): Détermine les règles d'écriture du HTML, la façon dont un agent utilisateur doit la traiter, les règles d'interactions entre les éléments etc. **Elle présente concrètement une machine état pour la construction de l'arbre HTML (le DOM). C'est la section qui nous intéresse le plus**. 
	- [La syntaxe XML](https://html.spec.whatwg.org/multipage/xhtml.html#the-xhtml-syntax): Détermine la façon dont un document HTML peut intégrer du XML. En effet, les deux peuvent se mélanger, comme on le verra plus tard dans ces articles. Lien : [Spécification du XML](https://www.w3.org/TR/xml/).
	- [Le Rendu](https://html.spec.whatwg.org/multipage/rendering.html#rendering): Détermine la façon dont un document HTML peut intégrer du CSS. Il existe un [*draft* de spécification CSS](https://drafts.csswg.org/css2/).

Pour terminer, il y a la section [Obsolete features](https://html.spec.whatwg.org/multipage/obsolete.html#obsolete) qui présente les fonctionnalités obsolètes, ainsi que diverses considérations qui ne nous intéressent pas.

La section la plus intéressante pour nous est donc celle qui concerne la syntaxe du HTML, et notamment le parsing.

## Le parsing HTML

Comme exposé plus haut, la section `Syntaxe HTML` est celle qui permet aux agents utilisateur d'implémenter la construction de l'arbre, et de rendre ce dernier sous la forme d'un objet : le **DOM**.

Pour faire simple, on peut voir le processus de parsing comme suit :

![Schéma du parsing](content/explorer_la_spec_html5_the_cool_way/schema_parsing.png)

Nous n'allons pas détailler l'ensemble de ces processus, mais plutôt aborder certains détails de ces processus. Ce n'est pas idéal car il y aura forcément des raccourcis, mais le but est de s'armer théoriquement pour tester des mutations, pas de faire notre propre navigateur.

> [!note]
> Nous utiliserons massivement l'outil [Dom-Explorer](https://yeswehack.github.io/Dom-Explorer/#eyJpbnB1dCI6IiIsInBpcGVsaW5lcyI6W3siaWQiOiJ2MzFtYjJ2YyIsIm5hbWUiOiJEb20gVHJlZSIsInBpcGVzIjpbeyJuYW1lIjoiRG9tUGFyc2VyIiwiaWQiOiI3YzNzZmlhayIsImhpZGUiOmZhbHNlLCJza2lwIjpmYWxzZSwib3B0cyI6eyJ0eXBlIjoidGV4dC9odG1sIiwic2VsZWN0b3IiOiJib2R5Iiwib3V0cHV0IjoiaW5uZXJIVE1MIiwiYWRkRG9jdHlwZSI6dHJ1ZX19XX1dfQ==) pour illustrer nos propos comme ci-dessous et tester différentes choses.
> Il s'agit d'une excellentissime application qui permet d'illustrer l'arbre HTML résultant d'une entrée utilisateur avec certains attributs utiles lors de l'exploration des mutations.
> Nous reviendrons sur cet outil plus en détail lors de l'article concernant la méthodologie et les outils, disponible ici [METTRE LIEN ARTICLE QUAND TERMINE].

### **Le flux d'octets**

Correspond aux octets bruts. Le parseur doit déterminer le jeu de caractères (`utf-8`, `utf-16[le/be]` etc). L'encodage permet de déterminer ce "qu'est" un octet ou une suite d'octets.

Par exemple, prenons le deux octets suivants : `\x3c\x00`. Selon que l'encodage soit en `utf-8` ou en `utf-16le`, la résultat est sensiblement différent. Tous deux donneront le caractère `<`, mais dans le cadre de l'`utf-8` ce résultat est calculé à partir de 1 seul octet - `\x3c` - tandis que pour l'`utf-16le`, c'est sur **deux octets**. Ainsi dans le cadre de l'`utf8`, le second octet `\x00` sera vu comme un octet à part entière. 

> [!note]
> L'`utf-16[be/le]` est cependant un cas particulier concernant les caractères ASCII (de 0 à 127) : c'est l'un des seuls jeux de caractères autorisés (avec l'`utf-32`) dont la plage ASCII est encodée différemment (sur deux octets au lieu d'un seul, quatre octets pour l'`utf-32`).

Mais comment le parseur détermine-t-il le jeu de caractères à appliquer au flux d'octets ?

Ceci est déterminé dans la section suivante : [Déterminer l'encodage des caractères](https://html.spec.whatwg.org/multipage/parsing.html#determining-the-character-encoding).

Synthétiquement, l'algorithme proposé possède trois niveaux de certitude : `tentative`, `certain`, ou `irrelevant`.

Ces niveaux de certitude sont déterminés à partir des différentes sources d'informations :

1. Le `BOM` pour *Byte Order Mark*, est une suite d'octets situés à l'index 0 du flux de caractères.

| BOM            | Encodage                                               | **Encodage final** |
| -------------- | ------------------------------------------------------ | ------------------ |
| 0xEF 0xBB 0xBF | [UTF-8](https://encoding.spec.whatwg.org/#utf-8)       | `utf-8`            |
| 0xFE 0xFF      | [UTF-16BE](https://encoding.spec.whatwg.org/#utf-16be) | `utf-8`            |
| 0xFF 0xFE      | [UTF-16LE](https://encoding.spec.whatwg.org/#utf-16le) | `utf-8`            |

2. Une préférence utilisateur ([`x-user-define`](https://encoding.spec.whatwg.org/#x-user-defined)) : selon le standard, l'encodage sera - de toutes façons - [windows-1252](https://encoding.spec.whatwg.org/#windows-1252)
3. Les 1024 premiers octets ou le nombre d'octets lus dans un laps de temps de 500ms. Ceci soit dans une pré-analyse du flux, soit lors de la lecture d'un des éléments suivant de cette liste. Cela permet notamment de déterminer le jeu de caractère à l'aide de l'élément `meta` avec l'attribut `charset`.
4. L'en-tête HTTP `Content-type`
5. L'encodage du parent du `Document` actuellement parsé
6. L'historique lié à la page (*ie.*, si le site a été visité auparavant, on peut prendre l'encodage déterminé pour lui comme encodage pour le document courant)
7. Auto-détection, autre façon de déterminer l'encodage, par exemple *via* la localisation de la page
8. Une valeur par défaut déterminée par l'implémentation. C'est l'`utf-8` qui est fortement suggéré.

Une fois le jeu de caractère déterminé, le décodeur avec les options idoines est utilisé pour interpréter les caractères présents.

Il est à noter que les jeux de caractères disponibles sont limités à une liste précise, parmi lesquels : [UTF-8](https://encoding.spec.whatwg.org/#utf-8), [ISO-8859-2](https://encoding.spec.whatwg.org/#iso-8859-2), [ISO-8859-7](https://encoding.spec.whatwg.org/#iso-8859-7), [ISO-8859-8](https://encoding.spec.whatwg.org/#iso-8859-8), [windows-874](https://encoding.spec.whatwg.org/#windows-874), [windows-1250](https://encoding.spec.whatwg.org/#windows-1250), [windows-1251](https://encoding.spec.whatwg.org/#windows-1251), [windows-1252](https://encoding.spec.whatwg.org/#windows-1252), [windows-1254](https://encoding.spec.whatwg.org/#windows-1254), [windows-1255](https://encoding.spec.whatwg.org/#windows-1255), [windows-1256](https://encoding.spec.whatwg.org/#windows-1256), [windows-1257](https://encoding.spec.whatwg.org/#windows-1257), [windows-1258](https://encoding.spec.whatwg.org/#windows-1258), [GBK](https://encoding.spec.whatwg.org/#gbk), [Big5](https://encoding.spec.whatwg.org/#big5), [ISO-2022-JP](https://encoding.spec.whatwg.org/#iso-2022-jp), [Shift_JIS](https://encoding.spec.whatwg.org/#shift_jis), [EUC-KR](https://encoding.spec.whatwg.org/#euc-kr), [UTF-16BE](https://encoding.spec.whatwg.org/#utf-16be), [UTF-16LE](https://encoding.spec.whatwg.org/#utf-16le), [UTF-16BE/LE](https://encoding.spec.whatwg.org/#utf-16be-le), et [x-user-defined](https://encoding.spec.whatwg.org/#x-user-defined). 

### La tokenisation et la construction de l'arbre : explications rapides

C'est lors des étapes de tokenisation et de construction que commence véritablement le travail.

Pour simplifier, le tokeniseur va émettre des tokens qui seront consommés par l'étape de construction de l'arbre. Ainsi, les deux opérations coexistent, chaque token émis devant être immédiatement consommé par l'étape de construction (c'est moi qui souligne) :

> **When a token is emitted, it must immediately be handled by the [tree construction](https://html.spec.whatwg.org/multipage/parsing.html#tree-construction) stage.** The tree construction stage can affect the state of the tokenization stage, and can insert additional characters into the stream. [...]

En somme, nous avons un jeu de deux programmes dont les influences sont bi-directionnelles. Les influences sont en l'espèce des variables tenues à jour par l'agent utilisateur. Nous les verrons un peu plus loin.

- **Le tokeniseur** 

C'est essentiellement une machine à états qui va consommer des caractères. Pour rappel, ces caractères sont déjà encodés lors de l'étape que nous avons décrite précédemment. Les états peuvent être influencés par le mode d'insertion ([*insertion mode*](https://html.spec.whatwg.org/multipage/parsing.html#insertion-mode)) ainsi que la "Pile des éléments ouverts" ([*stack of open elements*](https://html.spec.whatwg.org/multipage/parsing.html#the-stack-of-open-elements)) et d'autres choses.

Le tokeniseur retourne donc des tokens. Ces tokens sont catégorisés par type : `DOCTYPE`, `start tag`, `end tag`, `comment`, `character`, `end-of-file`.

- **Le constructeur**

L'étape de construction quant à elle consomme les tokens émis par l'étape de tokénisation.

La construction est non seulement influencée par le mode d'insertion, c'est même la principale variable utilisée, mais également par la liste des éléments de formatage actifs, l'état dans lequel on se trouve et tout un tas de choses qu'il serait trop long de parcourir et expliquer.

Ceci étant dit, le constructeur consomme certes des tokens, mais ce qui en résulte  c'est le DOM, auquel est rattaché un objet nommé `Document`, créé lors de l'instanciation du parseur.

Ce DOM est constitué de nœuds (*node*) reliés ensemble. Ces nœuds ont diverses propriétés, que nous verrons plus loin dans cet article, et ces propriétés peuvent changer "en cours de route". 

Ainsi un nœud qui pouvait avoir à un instant T telle ou telle propriété peut, suite à une opération donnée, en posséder de nouvelles, en être dépossédée d'autres etc. Autre chose, un nœud qui pouvait être relié à un autre peut également ne plus l'être suite à telle ou telle opération.

En somme, c'est ici que se trouve la nature dynamique du DOM.

### Présentations de quelques variables utilisées par les agents utilisateurs

Nous allons voir les 5 variables utiles pour notre affaire : états, modes d'insertion, pile des éléments ouverts et liste des éléments de formatage actifs.

- [**Les États :**](https://html.spec.whatwg.org/multipage/parsing.html#tokenization)

 Il y a de très nombreux états. Parmi eux, on peut trouver le *Data State*, qui est l'état initial, le *RAWTEXT state* etc. Les états permettent au programme de tokenisation de consommer un token (en général un caractère) selon les règles dudit état et du résultat des opérations menées au sein de cet état. 

-  [**Les Modes d'insertion :**](https://html.spec.whatwg.org/multipage/parsing.html#insertion-mode)

 Les modes d'insertion servent à contrôler la construction de l'arbre. 

 Il y a également de très nombreux modes. Le mode d'insertion initial est tout simplement *initial*. On trouve également le mode d'insertion [*in body*](https://html.spec.whatwg.org/multipage/parsing.html#parsing-main-inbody), [*in table*](https://html.spec.whatwg.org/multipage/parsing.html#parsing-main-intable) et l'un des plus intéressants concernant notre cas, le mode d'insertion [*in foreign content*](https://html.spec.whatwg.org/multipage/parsing.html#parsing-main-inforeign). Nous reviendrons sur ce dernier mode d'insertion en détail ici [INSERER SECTION CORRESPONDANTE].

-  [**La pile des éléments ouverts :**](https://html.spec.whatwg.org/multipage/parsing.html#the-stack-of-open-elements)

 C'est une pile où sont entreposés les éléments dit "ouverts", c'est à dire n'étant pas encore fermés... Belle lapalissade afin de signifier que cette pile est en quelque sorte un registre temporaire du DOM en cours de construction, dont les différents éléments à des instants T vont être poussés sur la pile, ou dépilés, selon des règles déterminées par l'état de la machine à états et bien sûr selon le mode d'insertion.

 Mais pas seulement ! En sus de cela, les éléments eux-même ont des règles qui leur sont propres (cf. lien ci-dessus), ce qui rend le traitement des différents éléments particulièrement complexe. Parfois, on peut se demander si l'exception ne serait la norme dans ce standard :D .

![comportement spéciaux html](content/explorer_la_spec_html5_the_cool_way/special_html.png)


 Le premier élément de la pile des éléments ouverts sera **toujours** l'élément `html`, et ne sera dépilé qu'une fois le parsing terminé.

-  **[Liste des éléments de formatage actifs :](https://html.spec.whatwg.org/multipage/parsing.html#the-list-of-active-formatting-elements)**

> [!warning]
> Cette section est particulièrement difficile à comprendre je trouve. Aussi je prends des raccourcis, et il est fort possible que des erreurs s'y trouvent malgré mes efforts.

Cette liste permet de prendre en charge les éléments qui sont mal imbriqués entre-eux *en les reconstruisant*. En effet, le standard HTML actuel - à la différence du XML - tolère des erreurs dans son écriture.

Les éléments de formatage sont dans cette liste : [a](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-a-element), [b](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-b-element), [big](https://html.spec.whatwg.org/multipage/obsolete.html#big), [code](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-code-element), [em](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-em-element), [font](https://html.spec.whatwg.org/multipage/obsolete.html#font), [i](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-i-element), [nobr](https://html.spec.whatwg.org/multipage/obsolete.html#nobr), [s](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-s-element), [small](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-small-element), [strike](https://html.spec.whatwg.org/multipage/obsolete.html#strike), [strong](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-strong-element), [tt](https://html.spec.whatwg.org/multipage/obsolete.html#tt), et [u](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-u-element). 

Il y a également ce que *whawg* nomme des "marqueurs" (*markers*), c'est à dire une liste d'éléments qui permettent de "bloquer" à un nœud l'opération de reconstruction ou de réagencement. Ou dit autrement, à leur manière, cela permet d'empêcher la "fuite" de certains éléments en dehors de ces marqueurs (par exemple la fuite en dehors d'un élément `template`), faisant de ces derniers la "racine" à partir de laquelle les opérations induites peuvent être menées.

Règle supplémentaire, il ne peut y avoir plus trois éléments identiques de formatage après un marqueur donné. Par identique, il faut entendre : même nom d'élément, même attributs, même namespace.

Je dois avouer que je trouve ce passage particulièrement ardu. Comme pour l'ensemble de ces articles, il faut prendre ce que j'écris avec des pincettes.

La liste des éléments de formatage actif semble être utile pour au moins deux opérations, comme nous l'avons évoqué dans les paragraphes précédents :

- Un algorithme de réagencement du HTML nommé [*Adoption Agency Algorithm*](https://html.spec.whatwg.org/multipage/parsing.html#adoptionAgency) (AAA à partir de maintenant)
- Un algorithme de reconstruction du HTML, détaillé au sein de la section [*Liste des éléments de formatage actifs*](https://html.spec.whatwg.org/multipage/parsing.html#the-list-of-active-formatting-elements)

Pour mieux saisir ce que c'est exactement, prenons l'exemple mis en avant par whatwg.

Le HTML suivant est mal formé :

```html
<p>1<b>2<i>3</b>4</i>5</p>
```

On voit que l'élément `b` est fermé avant l'élément `i` qui se trouve pourtant après `b`.

À ce moment nous avons :

| Pile des éléments ouverts    | Liste des éléments de formatage actifs |
| ----------------------------- | -------------------------------------- |
| `html`, `body`, `p`, `b`, `i` |  `b`, `i`                              |

L'agent utilisateur va donc reconstruire un arbre valide, qui deviendra *in fine* :

{{ domexplorer(id="#eyJpbnB1dCI6IjxwPjE8Yj4yPGk+MzwvYj40PC9pPjU8L3A+IiwicGlwZWxpbmVzIjpbeyJpZCI6Im1pNjI1OWZxIiwibmFtZSI6IkRvbSBUcmVlIiwicGlwZXMiOlt7Im5hbWUiOiJEb21QYXJzZXIiLCJpZCI6ImcwMXF0ZjlqIiwiaGlkZSI6ZmFsc2UsInNraXAiOmZhbHNlLCJvcHRzIjp7InR5cGUiOiJ0ZXh0L2h0bWwiLCJzZWxlY3RvciI6ImJvZHkiLCJvdXRwdXQiOiJpbm5lckhUTUwiLCJhZGREb2N0eXBlIjp0cnVlfX1dfV19") }}

**Un nouvel élément `i` a été créé !!**

Est-ce de la magie ?

![marabout](content/explorer_la_spec_html5_the_cool_way/marabout.png)

Malheureusement, non.

Pour comprendre, nous devons prendre en compte 4 choses : 
- **Le flux de tokens** et la tête de lecture s'y référant (en somme, l'élément qui est lu à un instant T)
- **Les listes des éléments ouverts** et **des éléments de formatage actifs**
- **Le DOM**, l'arbre résultant des diverses opérations.

1. Jusqu'à l'élément de texte `3`, les éléments sus-mentionnés ont les valeurs suivantes :

**Flux de tokens & DOM Résultant**

{{ domexplorer(id="eyJpbnB1dCI6IjxwPjE8Yj4yPGk+MyIsInBpcGVsaW5lcyI6W3siaWQiOiJtaTYyNTlmcSIsIm5hbWUiOiJEb20gVHJlZSIsInBpcGVzIjpbeyJuYW1lIjoiRG9tUGFyc2VyIiwiaWQiOiJnMDFxdGY5aiIsImhpZGUiOmZhbHNlLCJza2lwIjpmYWxzZSwib3B0cyI6eyJ0eXBlIjoidGV4dC9odG1sIiwic2VsZWN0b3IiOiJib2R5Iiwib3V0cHV0IjoiaW5uZXJIVE1MIiwiYWRkRG9jdHlwZSI6dHJ1ZX19XX1dfQ==") }}

| Pile des éléments ouverts    | Liste des éléments de formatage actifs |
| ----------------------------- | -------------------------------------- |
| `html`, `body`, `p`, `b`, `i` |  `b`, `i`                              |

2. Mais lorsque le parseur arrive à l'élément `</b>`, l'AAA est invoqué :

> [!note]
> Je vais être honnête : j'ai du mal à le comprendre. Afin de mieux le saisir, il faudrait que  j'implémente cette portion de la spécification au sein d'un programme, mais j'hésite entre :
> 
> ![flemme](content/explorer_la_spec_html5_the_cool_way/flemme.png)
> 
> Nous nous contenterons donc du résultat de cet algo.

**Flux de tokens & DOM résultant**

{{ domexplorer(id="eyJpbnB1dCI6IjxwPjE8Yj4yPGk+MzwvYj4iLCJwaXBlbGluZXMiOlt7ImlkIjoibWk2MjU5ZnEiLCJuYW1lIjoiRG9tIFRyZWUiLCJwaXBlcyI6W3sibmFtZSI6IkRvbVBhcnNlciIsImlkIjoiZzAxcXRmOWoiLCJoaWRlIjpmYWxzZSwic2tpcCI6ZmFsc2UsIm9wdHMiOnsidHlwZSI6InRleHQvaHRtbCIsInNlbGVjdG9yIjoiYm9keSIsIm91dHB1dCI6ImlubmVySFRNTCIsImFkZERvY3R5cGUiOnRydWV9fV19XX0=") }}

| Pile des éléments ouverts | Liste des éléments de formatage actifs |
| -------------------------- | -------------------------------------- |
| `html`, `body`, `p`        |  `i`                                   |

On constate que la structure du DOM est inchangée mais que la liste des éléments de formatage actifs  contient désormais `i` seulement et que la pile des éléments ouverts ne possède plus que `html`, `body` et `p`. L'AAA a donc retiré ces éléments de cette pile, ils sont donc maintenant fermés.

3. On consomme le token suivant qui est l'élément de texte `4` :

Cet élément de texte va déclencher l'algorithme de reconstruction de l'élément de formatage actif `i` décrit dans la section que nous sommes en train d'étudier.

Cet algorithme **permet de créer un nouvel élément équivalent à un élément de la liste des éléments de formatage actifs si** (dans notre cas):

- La liste des éléments de formatage actifs n'est pas vide
- L'élément traité par l'algorithme n'est pas un marqueur (comme `template`) et n'est pas dans la pile des éléments ouverts
- Il n'a pas d'élément avant lui dans la liste des éléments de formatage actifs, auquel cas d'autres opérations sont menées.

**Flux de tokens & DOM résultant**

{{ domexplorer(id="eyJpbnB1dCI6IjxwPjE8Yj4yPGk+MzwvYj40IiwicGlwZWxpbmVzIjpbeyJpZCI6Im1pNjI1OWZxIiwibmFtZSI6IkRvbSBUcmVlIiwicGlwZXMiOlt7Im5hbWUiOiJEb21QYXJzZXIiLCJpZCI6ImcwMXF0ZjlqIiwiaGlkZSI6ZmFsc2UsInNraXAiOmZhbHNlLCJvcHRzIjp7InR5cGUiOiJ0ZXh0L2h0bWwiLCJzZWxlY3RvciI6ImJvZHkiLCJvdXRwdXQiOiJpbm5lckhUTUwiLCJhZGREb2N0eXBlIjp0cnVlfX1dfV19") }}

On constate qu'un nouvel élément `i` identique a été créé.

4. *In fine*, après avoir traité le reste des tokens :

**Flux de tokens & DOM résultant**

{{ domexplorer(id="#eyJpbnB1dCI6IjxwPjE8Yj4yPGk+MzwvYj40PC9pPjU8L3A+IiwicGlwZWxpbmVzIjpbeyJpZCI6Im1pNjI1OWZxIiwibmFtZSI6IkRvbSBUcmVlIiwicGlwZXMiOlt7Im5hbWUiOiJEb21QYXJzZXIiLCJpZCI6ImcwMXF0ZjlqIiwiaGlkZSI6ZmFsc2UsInNraXAiOmZhbHNlLCJvcHRzIjp7InR5cGUiOiJ0ZXh0L2h0bWwiLCJzZWxlY3RvciI6ImJvZHkiLCJvdXRwdXQiOiJpbm5lckhUTUwiLCJhZGREb2N0eXBlIjp0cnVlfX1dfV19") }}

**OUF !!**


### Les nœuds

Il me semble important d'aborder rapidement ce qu'est un nœud :

![](content/explorer_la_spec_html5_the_cool_way/noeud1.png)

La belle jambe...

Le plus simple nœud qu'on puisse trouver dans HTML est du type [`Node`](https://dom.spec.whatwg.org/#node).

D'autres type de nœuds étendent (au sens programmatique) `Node`. Ils sont déterminés par l'attribut `nodeType`.

Il y a 12 `nodeType`, et parmi eux :

  - [ELEMENT_NODE](https://dom.spec.whatwg.org/#dom-node-element_node) = 1;
  - [ATTRIBUTE_NODE](https://dom.spec.whatwg.org/#dom-node-attribute_node) = 2;
  - [TEXT_NODE](https://dom.spec.whatwg.org/#dom-node-text_node) = 3;
  - [CDATA_SECTION_NODE](https://dom.spec.whatwg.org/#dom-node-cdata_section_node) = 4;
  - `ENTITY_REFERENCE_NODE` = 5; // legacy
  - `ENTITY_NODE` = 6; // legacy
  - [PROCESSING_INSTRUCTION_NODE](https://dom.spec.whatwg.org/#dom-node-processing_instruction_node) = 7;
  - [COMMENT_NODE](https://dom.spec.whatwg.org/#dom-node-comment_node) = 8;
  - [DOCUMENT_NODE](https://dom.spec.whatwg.org/#dom-node-document_node) = 9;
  - [DOCUMENT_TYPE_NODE](https://dom.spec.whatwg.org/#dom-node-document_type_node) = 10;
  - [DOCUMENT_FRAGMENT_NODE](https://dom.spec.whatwg.org/#dom-node-document_fragment_node) = 11;
  - `NOTATION_NODE` = 12; // legacy

Le type de nœud qui va nous intéresser le plus est celui avec le `nodeType` 1 : [`Element`](https://dom.spec.whatwg.org/#element).

L'interface `Element` possède - entre autres - l'attribut suivant : [namespaceURI](https://dom.spec.whatwg.org/#dom-element-namespaceuri).

Le *namespace*.

En HTML on a trois namespaces considérés par le standard, `HTML`, `XML`, `MATHML`. Chacun de ces *namespaces* possède ses propres règles. Le HTML lui a la possibilité ***d'intégrer*** les *namespaces* étrangers que sont le XML et le MATHML.

Ces règles d'intégration sont déterminées par le fameux mode d'insertion : [`in foreign content`](https://html.spec.whatwg.org/multipage/parsing.html#parsing-main-inforeign)

Ce sera l'objet du prochain article !

![Mais non !](content/explorer_la_spec_html5_the_cool_way/mais_non.png)