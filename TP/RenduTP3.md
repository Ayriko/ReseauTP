# TP3
MOISKA Aymeric

## I/ARP

### 1.Echange ARP

Ping de John :
```bash
ping 10.3.1.12
PING 10.3.1.12 (10.3.1.12) 56(84) bytes of data.
64 bytes from 10.3.1.12: icmp_seq=1 ttl=64 time=0.558 ms
64 bytes from 10.3.1.12: icmp_seq=2 ttl=64 time=0.211 ms
64 bytes from 10.3.1.12: icmp_seq=3 ttl=64 time=0.324 ms
64 bytes from 10.3.1.12: icmp_seq=4 ttl=64 time=0.294 ms
^C
--- 10.3.1.12 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3112ms
rtt min/avg/max/mdev = 0.211/0.346/0.558/0.128 ms
```

Ping de Marcel :
```bash
ping 10.3.1.12
PING 10.3.1.12 (10.3.1.12) 56(84) bytes of data.
64 bytes from 10.3.1.12: icmp_seq=1 ttl=64 time=0.037 ms
64 bytes from 10.3.1.12: icmp_seq=2 ttl=64 time=0.073 ms
64 bytes from 10.3.1.12: icmp_seq=3 ttl=64 time=0.072 ms
64 bytes from 10.3.1.12: icmp_seq=4 ttl=64 time=0.044 ms
^C
--- 10.3.1.12 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3050ms
rtt min/avg/max/mdev = 0.037/0.056/0.073/0.016 ms
```

Commande pour regarder la table ARP :  `ip neigh show`

- Table de John :
```bash
ip n s
10.3.1.12 dev enp0s3 lladdr 08:00:27:b9:af:42 REACHABLE
10.3.1.1 dev enp0s3 lladdr 0a:00:27:00:00:07 REACHABLE
```

- Table de Marcel :
```bash
ip neigh show
10.3.1.1 dev enp0s3 lladdr 0a:00:27:00:00:07 REACHABLE
10.3.1.11 dev enp0s3 lladdr 08:00:27:69:1b:05 REACHABLE
```

Chacune des tables possèdent pour l'instant deux lignes après le ping, on constate respectivement l'ip de l'autre machine et l'ip de notre machine dû au ssh.  
L'adresse MAC de John est : **08:00:27:69:1b:05**
L'adresse MAC de Marcel est : **08:00:27:b9:af:42**

Prouver ses adresses MAC :
- pour voir la MAC de Marcel dans la table ARP de John, on peut utiliser cette commande connaissant l'ip de Marcel : `ip neigh show 10.3.1.12` 
```bash
ip neigh show 10.3.1.12
10.3.1.12 dev enp0s3 lladdr 08:00:27:b9:af:42 STALE
```
->  ce qui confirme que c'est bien son adresse MAC

- pour afficher la MAC de Marcel, depuis Marcel : `ip address show enp0s3`
```bash
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:b9:af:42 brd ff:ff:ff:ff:ff:ff
    inet 10.3.1.12/24 brd 10.3.1.255 scope global noprefixroute enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feb9:af42/64 scope link
       valid_lft forever preferred_lft forever
```
L'addresse MAC de marcel est à la seconde ligne.  

### 2. Analyse de trames


Pour supprimer les tables ARP : `sudo ip neigh flush all`  
On utilise ensuite la commande suivante pour enregistrer les trames passant par l'interface enp0s3 pendant les 20 prochaines secondes mais en n'écoutant pas le port 22. On y précise aussi le nom souhaité du fichier :  `sudo tcpdump -i enp0s3 -c 20 -w tp2_arp.pcapng not port 22`

[Trame capturée par Wireshark](wireshark/tp2_arp.pcapng)

## II/Routage

### 1. Mise en place du routage

```bash
[rocky@localhost ~]$ sudo firewall-cmd --list-all
[sudo] password for rocky:
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3 enp0s8
  sources:
  services: cockpit dhcpv6-client ssh
  ports:
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
[rocky@localhost ~]$ sudo firewall-cmd --get-active-zone
public
  interfaces: enp0s3 enp0s8
[rocky@localhost ~]$ sudo firewall-cmd --add-masquerade --zone=public
success
[rocky@localhost ~]$ sudo firewall-cmd --add-masquerade --zone=public --permanent
success
```

On constate que nous avons bien réussi à mettre en place le routage pour cette machine qui va nous servir de routeur.  
Nous passons ensuite à l'ajout des routes statiques nécessaires.

On va donc créer deux routes de manières définitives afin de pas avoir à les refaire au moindre reboot.

Je pensais qu'il fallait faire ça dans le routeur à chaque fichier route-<INTERFACE_NAME> et en écrivant ces lignes pour chaque interface : `10.3.1.0/24 via 10.3.1.254 dev enp0s3` et `10.3.2.0/24 via 10.3.2.254 dev enp0s8` mais avec un avant et après `ip route show`, j'ai l'impression que ces paramètres existaient déjà sans que j'ai à les ajouter.

```bash
ip route show
10.3.1.0/24 dev enp0s3 proto kernel scope link src 10.3.1.254 metric 100
10.3.2.0/24 dev enp0s8 proto kernel scope link src 10.3.2.254 metric 101
[rocky@localhost ~]$ sudo vim /etc/sysconfig/network-scripts/route-enp0s3
[sudo] password for rocky:
[rocky@localhost ~]$ sudo vim /etc/sysconfig/network-scripts/route-enp0s8
[rocky@localhost ~]$ ip route show
10.3.1.0/24 dev enp0s3 proto kernel scope link src 10.3.1.254 metric 100
10.3.2.0/24 dev enp0s8 proto kernel scope link src 10.3.2.254 metric 101
```

A ce moment là je ne pouvais donc toujours pas ping entre Marcel et John.  
J'ai alors décidé d'essayer, après avoir relu, de suivre l'étape Ajouter une route par défaut (du mémo). Je l'ai fait chez John et chez Marcel, en précisant à chacun l'adresse de passerelle correspond à l'adresse du routeur dans leur réseau respectif.  
On a donc chez John : 
```bash
sudo vim /etc/sysconfig/network
GATEWAY=10.3.1.254

$ sudo systemctl restart NetworkManager
```

Et chez Marcel :
```bash
sudo vim /etc/sysconfig/network
GATEWAY=10.3.2.254

$ sudo systemctl restart NetworkManager
```

A partir de là, les pings semblaient fonctionner. On va vérifier ça avec les trames.

### 2. Analyse de trames

Après un `sudo ip neigh flush all` et un ping de John vers Marcel :

- Table du routeur :
```bash
10.3.1.11 dev enp0s3 lladdr 08:00:27:69:1b:05 REACHABLE
10.3.1.1 dev enp0s3 lladdr 0a:00:27:00:00:07 DELAY
10.3.2.12 dev enp0s8 lladdr 08:00:27:b9:af:42 REACHABLE
```
- Table de John :
```bash
10.3.1.1 dev enp0s3 lladdr 0a:00:27:00:00:07 REACHABLE
10.3.1.254 dev enp0s3 lladdr 08:00:27:a1:af:fc REACHABLE
```
- Table de Marcel :
```bash
10.3.2.1 dev enp0s3 lladdr 0a:00:27:00:00:38 REACHABLE
10.3.2.254 dev enp0s3 lladdr 08:00:27:21:92:f3 REACHABLE
```

En sortant le 10.3.1.1 qui est lié à la connexion ssh, je pense qu'en partant de John il y a eu un échange ARP de lui vers le routeur (10.3.1.254) pour lui envoyer le paquet. Depuis le routeur, un échange ARP vers Marcel (10.3.2.12), recevant donc le ping. Pour le renvoyer, il y a échange ARP de marcel vers le routeur(10.3.2.254) et ce dernier réalise le dernier ARP pour avoir l'adresse IP de John (10.3.1.11) et lui faire parvenir le pong.  

[Trame du ping via routeur, capturée par Wireshark](wireshark/tp2_routage_marcel.pcapng)

**Tableau de suivi des trames du Ping**

| ordre | type trame  | IP source            | MAC source                  | IP destination            | MAC destination            | Info
|-------|-------------|----------------------|-----------------------------|---------------------------|----------------------------| ----
| 1     | Requête ARP | x                    | `john` `08:00:27:21:92:f3`  | x                         | Broadcast `FF:FF:FF:FF:FF` | Who has 10.13.2.12 ? Tell 10.3.2.254
| 2     | Réponse ARP | x                    | `marcel` `08:00:27:b9:af:42`| x                         | `john` `08:00:27:21:92:f3` | 10.3.2.12 is at 08:00:27:b9:af:42
| 3     | ICMP        | `Router` `10.3.2.254`| x                           | `Marcel` `10.3.2.12`      | x                          | Echo (ping) request
| 4     | ICMP        | `Marcel` `10.3.2.12` | x                           | `Router` `10.3.2.254`     | x                          | Echo (ping) reply

### 3. Accès Internet

#### Donner un accès internet à vos machines :

- Ajout de la carte NAT au routeur (visible via `ip a`)
```bash
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:fd:5a:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.0.4.15/24 brd 10.0.4.255 scope global dynamic noprefixroute enp0s9
       valid_lft 86374sec preferred_lft 86374sec
    inet6 fe80::11bf:801c:32ef:3453/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

- Ajout des routes par défaut à Marcel et John  
  John :
  ```bash
  vim /etc/sysconfig/network
  GATEWAY=10.3.1.254
  ```
  Marcel :
  ```bash
  vim /etc/sysconfig/network
  GATEWAY=10.3.2.254
  ```

Si on peut ping 8.8.8.8 dès deux côtés, on obtient des réponses. Ils ont bien accès à Internet.
```bash
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=246 time=16.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=246 time=14.9 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=246 time=13.1 ms
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 13.115/14.785/16.366/1.328 ms
```

- Fournir une adresse DNS et tester son fonctionnement  
On modifie le fichier resolv.conf et on y ajoute le DNS de cloudflare : 1.1.1.1.  
```bash
[rocky@localhost ~]$ sudo vim /etc/resolv.conf
[sudo] password for rocky:
[rocky@localhost ~]$  cat /etc/resolv.conf
nameserver 1.1.1.1
[rocky@localhost ~]$ dig gitlab.com

; <<>> DiG 9.16.23-RH <<>> gitlab.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4960
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;gitlab.com.                    IN      A

;; ANSWER SECTION:
gitlab.com.             77      IN      A       172.65.251.78

;; Query time: 15 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Tue Oct 11 14:22:40 CEST 2022
;; MSG SIZE  rcvd: 55
```
```bash
[rocky@localhost ~]$ ping gitlab.com
PING gitlab.com (172.65.251.78) 56(84) bytes of data.
64 bytes from 172.65.251.78 (172.65.251.78): icmp_seq=1 ttl=248 time=12.5 ms
64 bytes from 172.65.251.78 (172.65.251.78): icmp_seq=2 ttl=248 time=15.0 ms
^C
--- gitlab.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 12.501/13.740/14.980/1.239 ms
```

#### Analyse de trames

- Ping de John et capture  
Le ping et la capture ont été faite sur deux instances de John (une via virtualbox directement et l'autre via ssh)
```bash
[rocky@localhost ~]$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=246 time=15.7 ms
^[[A^[[A64 bytes from 8.8.8.8: icmp_seq=2 ttl=246 time=14.5 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=246 time=15.6 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=246 time=15.6 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 14.514/15.355/15.694/0.487 ms
```

```bash
[rocky@localhost ~]$ sudo tcpdump -i enp0s3 -c 10 -w tp2_routage_internet.pcap not port 22
dropped privs to tcpdump
tcpdump: listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10 packets captured
11 packets received by filter
0 packets dropped by kernel
```

| ordre | type trame  | IP source            | MAC source          | IP destination    | MAC destination     | Info
|-------|-------------|----------------------|---------------------|-------------------|---------------------| ----
| 1     | ICMP        | `John` `10.3.1.11`   | `08:00:27:69:1b:05` | `8.8.8.8`         | `08:00:27:a1:af:fc` | Echo (ping) request
| 2     | ICMP        | `8.8.8.8`            | `08:00:27:a1:af:fc` | `John` `10.3.1.11`| `08:00:27:69:1b:05` | Echo (ping) reply
....   

On observe que comparé aux pings entre John et Marcel, ceux-ci n'ont pas l'air de "passer" à travers le routeur même si c'est lui qui fournit Internet.

[Trame du ping Internet via routeur, capturée par Wireshark](wireshark/tp2_routage_internet.pcapng)

## DHCP

### 1/ Mise en place du serveur DHCP

- Configuration de John en DCHP

On installe d'abord le service :

```bash
sudo dnf install dhcp-server
sudo vim /etc/dhcp/dhcpd.conf
```
On y inscrit cela :
```bash
default-lease-time 900;
max-lease-time 10800;
ddns-update-style none;
authoritative;
option domain-name-servers 1.1.1.1;
subnet 10.3.1.0 netmask 255.255.255.0 {
  range dynamic-bootp 10.3.1.12 10.3.1.20;
  option routers 10.3.1.254; 
}

sudo systemctl enable --now dhcpd
```

Ensuite il faut ouvrir le port UDP 67.

```bash
sudo firewall-cmd --permanent --add-port=67/udp
```

- Création de bob et récupération d'une IP

On configure l'interface de Bob (ifcfg-enp0s3) avec :
```bash
NAME=enp0s8
DEVICE=enp0s8

BOOTPROTO=dhcp
ONBOOT=yes
```
Et on la redémarre.  
Avec cela il va récupérer automatiquement une IP via le DHCP après un reboot.

- Amélioration de la configuration du DHCP

Première partie déjà réalisé plus tôt, afin qu'il fournisse un serveur DNS et une route par défaut.  
Test de Bob :

Récupération d'une IP -> On peut le faire avec dhclient
```bash
[rocky@localhost ~]$ sudo dhclient
[sudo] password for rocky:
[rocky@localhost ~]$ ip a
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:fa:2f:44 brd ff:ff:ff:ff:ff:ff
    inet 10.3.1.12/24 brd 10.3.1.255 scope global dynamic noprefixroute enp0s3
       valid_lft 596sec preferred_lft 596sec
    inet 10.3.1.13/24 brd 10.3.1.255 scope global secondary dynamic enp0s3
       valid_lft 958sec preferred_lft 958sec
    inet6 fe80::a00:27ff:fefa:2f44/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

On voit qu'il a récupéré une seconde IP (10.3.1.13) en plus de celle attribuée automatiquement plus tôt (10.3.1.12).
On va pouvoir supprimer la première avec la commande suivante : `ip addr del 10.3.1.12/24 dev enp0s3` (coupure ssh incoming)  
Ping vers la passerelle :
```bash
[rocky@localhost ~]$ ping 10.3.1.254
PING 10.3.1.254 (10.3.1.254) 56(84) bytes of data.
64 bytes from 10.3.1.254: icmp_seq=1 ttl=64 time=0.532 ms
64 bytes from 10.3.1.254: icmp_seq=2 ttl=64 time=0.801 ms
^C
--- 10.3.1.254 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1063ms
rtt min/avg/max/mdev = 0.532/0.666/0.801/0.134 ms
```

On vérifie qu'il est obtenu une route par défaut :
```bash
[rocky@localhost ~]$ ip route show
default via 10.3.1.254 dev enp0s3
10.3.1.0/24 dev enp0s3 proto kernel scope link src 10.3.1.13
```

Il semble l'avoir, on confirme avec un ping vers le réseau 10.3.2.0/24
```bash
[rocky@localhost ~]$ ping 10.3.2.12
PING 10.3.2.12 (10.3.2.12) 56(84) bytes of data.
64 bytes from 10.3.2.12: icmp_seq=1 ttl=63 time=1.12 ms
64 bytes from 10.3.2.12: icmp_seq=2 ttl=63 time=1.36 ms
^C
--- 10.3.2.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.122/1.241/1.361/0.119 ms
```

Finalement on vérifie le serveur DNS.
```bash
[rocky@localhost ~]$ dig github.com

; <<>> DiG 9.16.23-RH <<>> github.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41240
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;github.com.                    IN      A

;; ANSWER SECTION:
github.com.             43      IN      A       140.82.121.3

;; Query time: 14 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Tue Oct 11 15:57:03 CEST 2022
;; MSG SIZE  rcvd: 55
```

```bash
[rocky@localhost ~]$ ping github.com
PING github.com (140.82.121.3) 56(84) bytes of data.
64 bytes from lb-140-82-121-3-fra.github.com (140.82.121.3): icmp_seq=1 ttl=247 time=21.1 ms
64 bytes from lb-140-82-121-3-fra.github.com (140.82.121.3): icmp_seq=2 ttl=247 time=23.5 ms
^C
--- github.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 21.087/22.288/23.490/1.201 ms
```

Bob a donc bien tout obtenu du serveur DHCP John lorsqu'il a fait récupéré une adresse IP.

### 2/ Analyse de trames

Réalisé avec Bob et John  
`sudo tcpdump -i enp0s3 -c 10 -w tp2_dhcp.pcap not port 22` chez John  
et  
`dhclient` chez Bob  

[Trame de la demande IP capturée par Wireshark](wireshark/tp2_dhcp.pcapng)

