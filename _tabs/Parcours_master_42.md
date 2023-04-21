---
title: Le Master à 42
order: 3
icon: fas fa-file
img_path: /assets/img/icons
pin: true
---

> 42 est une école avec une pédagogie particulière. Aussi me semble-t-il utile de spécifier la philosophie de l'école ainsi que le parcours prévu afin de réaliser le master en Administration des systèmes d'information, réseaux et sécurité.

## 42, c'est quoi ?

42 est une école d'informatique accessible à toutes et tous dès lors qu'on a 18 ans, peu importe son niveau d'étude. Après un concours d'un mois nommé "La piscine", commence la première partie du cursus : le tronc commun. 

Le tronc commun pose les bases qu'une développeuse ou un développeur se doit de posséder. Il est constitué de projets principalement en langage C (à l'exception notable du projet final, qui est une application web écrite en typescript), et brosse différents aspects de la programmation informatique.

Outre la programmation, c'est aussi l'occasion de découvrir d'autres domaines, et notamment l'administration système et réseaux ainsi que la virtualisation avec docker.

Une fois ce cercle terminé, nous avons la possibilité de continuer le cursus afin de réaliser une spécialisation.


## La spécialisation Administration des systèmes d'information, réseaux et sécurité

Pour ma part, j'ai commencé cette spécialisation, qui se terminera par l'obtention d'un équivalent master. Pour ce faire, je dois réaliser un certains nombres de projets que je vais décrire ci-après, ainsi qu'une expérience professionnelle (via stage ou alternance, ce que je souhaite dans l'idéal) et un mémoire à présenter à un jury d'expertes et experts dans leur domaines. 

Voici les projets déjà fait ou que je vais faire par la suite. Ils sont présentés selon leur domaine.

### Sécurité:
> La branche sécurité de 42 est composée de différents projets qui visent la plupart du
temps à pointer des failles de sécurité au sein d'un serveur, site web etc.

 - Snowcrash, Rainfall, Override : ce sont trois projet de type "box". Il s'agit
de trouver des failles au sein d'un serveur afin de valider un niveau et passer
au suivant (mauvaise configuration à une exploitation de binaire etc).

- ft_malcom: une introduction aux attaques de type "man in the middle"

- darkly: le top 10 OWASP vus sur ce projet.

- tinky-winkey: une introduction à windows avec la création d'un service qui fait
tourner un "keylogger" (un enregistreur de touches)

### Administration systèmes et réseaux :
> Cette branche vise à présenter et à apprendre à monitorer un réseau (du type d'un FAI par exemple), connaître en profondeur les protocoles réseaux (tcp ip par exemple), et diverses autres techniques en relation avec ce large domaine.

- Bgp At Doors of Autonomous Systems is Simple: créer à l'aide de GNS3 des réseaux de types VXLAN, BGP-EVPN.

- Inception-of-Things: introduction à kubernetes sous la perspective d'un développeur (intégration continue etc)

- Cloud-1: introduction aux serveurs cloud.

### Développement d'outils:
>Cette partie est spéciale car ce n'est pas une branche en soi. Cependant, elle est essentielle aux deux parties ci-dessus, en ceci qu'elle permet d'appréhender sous l'angle de la fabrication des concepts qui sinon seraient brossés seulement sous un angle pratique.

- ft_ping, ft_traceroute, ft_nmap : un trio de projets qui vise à comprendre de façon extrêmement fine le protocole tcp-ip et divers autres protocoles en recodant des programmes très fameux dans le monde de l'informatique et celui de la cybersécurité.

- taskmaster, matt-deamon: ce sont deux projets qui sont sensiblement les mêmes. Les deux visent à créer un manager de programme. Le premier, qui est déjà fait,est écrit en python, tandis que le second le sera obligatoirement en c++ et consistera à faire du programme un "daemon" (dans le sens linuxien).

### C'est tout ?

C'est déjà pas mal ! Mais surtout, je pense que la philosophie de 42 est pleinement embrassée si on se forme également à côté. Pour ma part - depuis la fin du tronc commun - je commence à développer certains de mes outils, surtout pour me faciliter ma vie, et je passe énormément de temps à pratiquer la sécurité informatique sur des plateformes comme root-me ou hack the box, ainsi que lors de ctf avec des camarades de l'asssociation de cybersécurité de mon école, dans laquelle je suis un membre actif.
