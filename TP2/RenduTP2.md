# TP Réseau 2
Aymeric MOISKA

## 1/ Setup IP

Réseau à au moins 38 places -> 38 est entre 32 et 64 soit 2^5 et 2^6. On garde donc le 64. 
32 - 6 = 26, pour accueillir au moins 38 clients il faut un masque 26 permettant un maximum de 62 IPs.  
L'IP d'un masque 26 est 255.255.255.192  

Nous avons pris respectivement 192.168.10.1/26 et 192.168.10.2/26 (moi).  
11000000.10101000.00001010.000000010   
11000000.10101000.00001010.000000000 -> 192.168.10.0/26   
11000000.10101000.00001010.001111111 -> 192.168.10.63/26  
L'adresse réseau est 192.168.10.0/26  
L'adresse broadcast est 192.168.10.63/26  

Commandes que nous utilisons en console :
```bash
netsh interface ipv4 set address name="Ethernet" static 192.168.10.2 255.255.255.192 192.168.10.63
```

Ping prouvant le connexion :
```bash
Envoi d’une requête 'Ping'  192.168.10.1 avec 32 octets de données :
Réponse de 192.168.10.1 : octets=32 temps=2 ms TTL=128
Réponse de 192.168.10.1 : octets=32 temps=2 ms TTL=128
Réponse de 192.168.10.1 : octets=32 temps=2 ms TTL=128
Réponse de 192.168.10.1 : octets=32 temps=2 ms TTL=128

Statistiques Ping pour 192.168.10.1:
    Paquets : envoyés = 4, reçus = 4, perdus = 0 (perte 0%),
Durée approximative des boucles en millisecondes :
    Minimum = 2ms, Maximum = 2ms, Moyenne = 2ms
```

[Trame du ping capturée par Wireshark](wireshark/ping1.pcapng)  
-> Le ping que l'on envoie est un ICMP de type 8, une requête echo.  
-> Le pong que l'on reçoie est un ICMP de type 0, une réponse echo.  


## 2/ ARP my bro  

Commande pour afficher la table ARP : `arp -a`  
On regarde au niveau de notre interface ethernet possédant notre IP : 192.168.10.2

```bash
Adresse Internet      Adresse physique      Type
  192.168.10.1          f4-39-09-75-4b-1f     dynamique
  192.168.10.63         ff-ff-ff-ff-ff-ff     statique
  192.168.137.2         f4-39-09-75-4b-1f     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
  224.0.0.251           01-00-5e-00-00-fb     statique
  224.0.0.252           01-00-5e-00-00-fc     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique
```

On retrouve l'IP de mon binôme à la première ligne, son adresse MAC est donc : f4-36-09-75-4b-1f.  

Pour l'adresse de la passerelle, on regarde l'interface lié à notre réseau wifi.

```bash 
Interface : 10.33.19.97 --- 0x5
  Adresse Internet      Adresse physique      Type
  10.33.19.254          00-c0-e7-e0-04-4e     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique
```
La ligne qui nous interesse est : `10.33.19.254          00-c0-e7-e0-04-4e     dynamique`  
L'adresse MAC de la passerelle est donc 00-c0-e7-e0-04-4e.

Commande pour vider la table ARP : `netsh interface ip delete arpcache`  
La commande `arp -d` ne fonctionnait pas, les paramètres étant à priori incorrect malgrè qu'elle ai fonctionnait pour mon binôme.

**Avant PING** (En conservant seulement les interfaces du wifi et de l'ethernet)

```
Interface : 10.33.19.97 --- 0x5
  Adresse Internet      Adresse physique      Type
  10.33.19.254          00-c0-e7-e0-04-4e     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique

Interface : 192.168.10.2 --- 0xb
  Adresse Internet      Adresse physique      Type
  192.168.10.63         ff-ff-ff-ff-ff-ff     statique
  224.0.0.22            01-00-5e-00-00-16     statique

```  
On peut voir qu'on ne possède plus d'informations sur l'autre binôme avec qui on partage un lan.   
De plus, malgré le fait qu'on est quelques secondes plus tôt supprimé la table, on a rapidement récupéré l'IP de la passerelle permettant de communiquer avec Internet.

**Après PING**

```bash
Interface : 10.33.19.97 --- 0x5
  Adresse Internet      Adresse physique      Type
  10.33.19.254          00-c0-e7-e0-04-4e     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
  224.0.0.251           01-00-5e-00-00-fb     statique
  224.0.0.252           01-00-5e-00-00-fc     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique
  255.255.255.255       ff-ff-ff-ff-ff-ff     statique

Interface : 192.168.10.2 --- 0xb
  Adresse Internet      Adresse physique      Type
  192.168.10.1          f4-39-09-75-4b-1f     dynamique
  192.168.10.63         ff-ff-ff-ff-ff-ff     statique
  224.0.0.22            01-00-5e-00-00-16     statique
  224.0.0.251           01-00-5e-00-00-fb     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique
```

On constate donc principalement le retour de la ligne (la première) contenant les informations de l'autre personne sur le réseau, que nous venons de ping.  

Observation Wireshark : 

[Trames ARP capturées par WireShark](wireshark/preuveARP2.pcapng)

Au moment où on éxécute le ping on peut apercervoir un protocol ARP précédant les protocoles ICMP. L'adresse de la source de la requête ARP broadcast est l'adresse MAC de notre PC (version du constructeur) avec pour destinataire l'adresse broadcast. Ceci a pour but de demander l'origine de l'IP à l'ensemble des membres du réseau. Sur la ligne d'en dessous, on constate que le membre possédant l'adresse nous répond via le même protocole mais en ARP reply. La source est son adresse MAC et nous sommes le destinataire. 
Le ping peut maintenant se dérouler avec succès.
On peut aussi voir à la fin du ping, que le protocole ARP prend place dans le sens inverse. L'autre membre a pu profiter du ping pour apprendre notre adresse IP pour ensuite récupérer notre adresse MAC.

## 3/ DHCP you too my bro

[Trames DCHP capturées par WireShark](wireshark/dhcp.pcapng)  

On peut observer sous la colonne "info" 4 trames différentes : Discover, Offer, Request et ACK (pour acknowledge)
- Discover : La source est notre PC qui ne possède pas encore d'adresse IP, la destination est l'adresse Broadcast.
- Offer : La source est la passerelle contenant plusieurs infos dont une IP qu'elle propose via la broadcast (la destination). Dans les options de la trame on voit aussi qu'il envoie le bail DHCP (option 51) parmis d'autres informations.
- Request : Notre PC ayant reçu l'offre via le broadcast fait une requête pour l'IP proposée (on peut la voir dans les options de la trame (option 50)) qu'il envoie vers l'adresse broadcast. Le serveur DHCP (ou notre box ici) captera le message. 
- Ack : La passerelle renvoie à travers la broadcast qu'il a accepté/ pris connaissance de la nouvelle IP pour ce membre du réseau. 

Identifiez dans ces 4 trames les informations 1, 2 et 3 dont on a parlé juste au dessus :
- L'ip est retrouvable dès la trame Offer et on la retrouve dans la trame Request.
- La passerelle partage l'adresse IP de la broadcast dans la trame Offer et Ack. (option 28)
- Idem pour l'adresse d'un domain name server. (option 6)

## 4/ Avant-goût TCP et UDP

[Trames UDP, request et reply lors d'une vidéo youtube](wireshark/ytb.pcapng)  
Au moment où on lance une vidéo youtube, wireshark observe un immense paquet de trames UDP presque instantanément.   
On constate que notre PC se connecte en destination à une MAC (première ligne), en regardant les détails on voit qu'elle est lié à SFR, mon opérateur -> SFR_52:e7:10(60:35:c0:52:e7:10)  
On voit aussi qu'on se connecte via le port 443, représantant les connexions au trafic web sécurisé (https)