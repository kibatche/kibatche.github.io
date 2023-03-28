---
title: pyjam.automatic, déploiement automatisé d'un tunnel pyjam.as
categories:
- Programme
- web
tags:
- programme
- python
- web
---

## Présentation

Petite présentation d'un programme très simple qui permet de déployer un tunnel [pyjam.as](https://tunnel.pyjam.as/) automatiquement.

Ngrok étant devenu de plus en plus restrictif sur son utilisation gratuite (avec notamment une _warning page_ qui bloque tout bot voulant venir sur notre app locale), pyjam.as est une alternative très intéressante pour créer un tunnel entre une application locale et internet.

En effet, dans de nombreuses situations (pour ma part principalement des challenges root-me et des machines sur hack the box), nous avons besoin de faire en sorte qu'un payload quelconque soit accessible via internet. C'est par exemple le cas lorsqu'on souhaite opérer une redirection vers notre app malveillante après une xss.

Ce programme très simple est écrit en python3.


## Usage

```bash
pyjam.py -h
usage: pyjam.py [-h] (-p PORT | -d | -u)

A python program to automate the creation of a pyjam.as tunnel.

options:
  -h, --help            show this help message and exit
  -p PORT, --port PORT  The port to bind to. If a tunnel already exists, it will be deleted.
  -d, --down            Down the tunnel
  -u, --up              Up the tunnel
```

L'outil est disponible ici : https://github.com/kibatche/pyjam.automatic
