# TCP, UDP et services réseau

## First steps

On utilise netstat pour observer les logiciels utilisant x tunnels et récupérer l'IP afin de la filtrer sur Wireshark.
`netstat -n -p x -b`

- Première application : Steam -> TCP

`netstat -n -p tcp -b`

```bash
TCP    10.33.19.97:5324       185.25.182.76:443      ESTABLISHED
 [Steam.exe]
TCP    10.33.19.97:5497       185.25.182.35:80       ESTABLISHED
 [Steam.exe]
TCP    10.33.19.97:5498       185.25.182.4:80        ESTABLISHED
 [Steam.exe]
.....
```

[Capture de la trame Wireshark preSteam](wireshark/pre_capture_steam.pcapng)

Dans Wireshark on voit plusieurs SYN au réseau lié à steam et les SYNACK et ACK qui s'en suivent. Mon steam a aussi lancé une mise à jour
d'un jeu juste à la suite, on observe qu'il a rapidement commencé à me PUSH et ACK toutes les données.
Pour Steam j'ai donc 3 connexions à 3 IPs partageant le même port de serveur, elles représentent les tunnels ouverts pour télécharger :  
(un autre test a montré qu'il pouvait en avoir bien plus que 3)  
IP : 185.25.182.36  
IP : 185.25.182.35  
IP : 185.25.182.4  
Port : 80  
Ma machine a ouvert trois ports locaux pour s'y connecter :  
Port : 5334  
Port : 5335  
Port : 5336  

On peut aussi comprendre que la première ligne netstat avec l'IP : 185.25.182.76 avec le port 443 correspond à l'IP générale de Steam.  
Pour lui notre port local est le 5554.  
Et donc que le port 80 semble être dédié aux téléchargements.  

[Capture de la trame Wireshark Steam](wireshark/capture_steam.pcapng)

- Seconde application : Twitch - TCP

Au delà de des connexions liés au navigateur web, on s'aperçoit via wireshark que les adresses envoyant le plus de données lorsque l'on est  
Twitch ressemblent à ça :  
```bash
TCP    [2a02:8428:f9d3:701:c12f:7490:aec1:2acf]:18988  [2a00:1450:4007:807::2003]:443  ESTABLISHED
 [chrome.exe]
```
C'est la première connexion dans "l'ordre" des ports ouverts au besoin quand on lance twitch donc on peut imaginer que c'est l'initiale.  
Les suivantes sont probablement utilisés pour les multiples transferts de données.   
Comme pour steam, on voit bcp de push/ack et ainsi de suite.  
Twitch a donc comme ip et port : [2a00:1450:4007:807::2003]:443  
Quant à nous : [2a02:8428:f9d3:701:c12f:7490:aec1:2acf]:18988  

[Capture de la trame Wireshark Twitch](wireshark/capture_twitch.pcapng)

- Troisième application : OneNote

Première connexion à OneNote :
```bash
TCP    192.168.1.57:19075     52.109.32.24:443       ESTABLISHED
 [onenoteim.exe]
```
C'est également une connexion parmi plusieurs autres mais c'est apparement la première au lancement du logiciel OneNote.  
192.168.1.54 est mon IP et le port ouvert est 19075.  
Pour le logiciel, 52.109.32.24 est son IP et comme les autres on voit qu'il est en https dû au port 443.

[Capture de la trame Wireshark OneNote](wireshark/capture_onenote.pcapng)

- Quatrième application : Foundry Virtual Tabletop

- Cinquième applicationn : Discord

De la même façon, j'oberve une première connexion active :
```bash
TCP    192.168.1.57:5763      162.159.135.234:443    ESTABLISHED
 [Discord.exe]
```

Mon adresse IP n'a pas changée, seulement le port qui est ouvert ici : 5736.  
L'addresse IP utilisée par discord est 169.159.135.234 et son port, idem 443.  

[Capture de la trame Wireshark Discord](wireshark/capture_discord.pcapng)

## Mise en place

### 1. SSH

La machine utilisée est John du précédent TP.

On se rend vite compte que tout se fait en TCP, c'est logique quand on y pense d'utiliser une connexion sécurisée  
quand on se connecte à distance sur une machine.  

[Capture de la trame Wireshark SSH](wireshark/capture_discord.pcapng)

J'ai le three-way-handshake, de la communication etc.. mais pas de Fin-Ack lorsque je coupe le ssh.  
J'imagine que le RST-ACK le contient, vu qu'il était toujours à la fin dès que je testais.

Ce qu'on trouve avec la commande `ss` :
```bash
tcp      ESTAB     0         52                             10.3.1.11:ssh             10.3.1.1:webobjects
```

Le port est renseigné en "ssh" mais on sait qu'il s'agit du port 22.

Ce qui confirme que le ssh se fait bien en TCP.

### 2. NFS (reste à faire lundi 18)

### 3. DNS

Commandes utilisées pour la capture :  
`sudo tcpdump -i enp0s3 -c 20 -w trafic_dns.pcapng not port 22` sur john en ssh  
`dig gitlab.com` chez John directement via VBox.  

[Capture de la trame Wireshark DNS](wireshark/capture_dns.pcapng)  

L'adresse IP du serveur DNS est 1.1.1.1 et son port est 53..  




