# TP3 - Attaques réseau : ARP, DNS, TCP, qui croit qui ?

* **Environnement :** Machine virtuelle Kali Linux & Laboratoires Docker SEED (`10.9.0.0/24`)
* **Référentiel MITRE ATT&CK :** 
  * `T1557.002` - ARP Cache Poisoning & `T1040` - Network Sniffing (*Credential Access, Collection*)
  * `T1557` - Adversary-in-the-Middle (*Credential Access, Collection*)
  * `T1499.001` - OS Exhaustion Flood (*Impact*)

---

## Exercice 1 : ARP Cache Poisoning & MITM

### 1. Ce que j'ai fait
* **Cartographie des cibles :** Relevé des adresses IP et MAC réelles des conteneurs actifs du labo (`ip -br link show eth0`) afin de les intégrer en variables dans mes scripts Python Scapy :
  * **Victime A (`A-10.9.0.5`) :** `26:29:14:2a:58:c0`
  * **Victime B (`B-10.9.0.6`) :** `72:40:be:4c:f7:7d`
  * **Attaquant M (`M-10.9.0.105`) :** `96:99:e9:ae:bb:da`
* **Tests d'empoisonnement (Task 1) :** Exécution des trois variantes de paquets ARP forgés vers `A-10.9.0.5` en usurpant l'IP `10.9.0.6` :
  * **Variante A :** `ARP Request` (`op=1`)
  * **Variante B :** `ARP Reply` (`op=2`) après purge du cache (`ip -s -s neigh flush all`)
  * **Variante C :** `Gratuitous ARP` en broadcast (`ff:ff:ff:ff:ff:ff`) après purge du cache
* **Interception Man-in-the-Middle (Task 2) :** Lancement d'un script Python en boucle (envoi toutes les 2 secondes) empoisonnant simultanément les caches de A et de B, activation du routage IP sur l'attaquant (`net.ipv4.ip_forward=1`) et capture d'une session **Telnet** dans **Wireshark**.

### 2. Ce que j'ai observé
* **Variante A (`ARP Request`) :** La table ARP de la victime (`arp -n`) associe immédiatement l'IP `10.9.0.6` à l'adresse MAC de l'attaquant (`96:99:e9:ae:bb:da`), même lorsque le cache initial est totalement vide.
* **Variantes B (`ARP Reply`) et C (`Gratuitous ARP`) :** Lorsque le cache ARP de `A` est vidé au préalable, ces deux paquets sont rejetés et `arp -n` reste vide.
* **Interception Wireshark :** Lors de la connexion Telnet de `10.9.0.5` vers `10.9.0.6`, la fonctionnalité *Follow TCP Stream* de Wireshark affiche en clair la totalité de la session, révélant l'identifiant (`seed`) et le mot de passe (`dees`).

### 3. Ce que j'ai compris & Questions de débriefing
* **Confiance aveugle d'ARP :** Le protocole ARP est sans état (*stateless*) et ne possède aucun mécanisme d'authentification. Par mesure de protection et d'optimisation (`arp_accept = 0`), le noyau Linux moderne ignore les réponses ARP non sollicitées si l'adresse IP cible n'existe pas déjà dans son cache, mais il enregistre systématiquement l'émetteur d'une requête ARP légitime.
* **Qu'est-ce qui aurait empêché la Task 2 de montrer le mot de passe ?**  
  L'utilisation d'un protocole chiffré de bout en bout comme **SSH** à la place de Telnet. L'attaque ARP MITM aurait toujours permis d'intercepter les trames sur le réseau local, mais la charge utile (identifiant, mot de passe, commandes) aurait été chiffrée et illisible.

---

## Exercice 2 : DNS Cache Poisoning

### 1. Ce que j'ai fait
* Identification de l'interface réseau bridge du laboratoire Docker (`br-d9dafec411b3` sur le réseau `10.9.0.1/24`).
* Lancement du script Python Scapy (`dns_sniff_spoof.py`) en écoute sur le port UDP `53` pour intercepter les requêtes DNS récursives sortantes émises par le serveur DNS local (`10.9.0.53`).
* Forgerie d'une réponse DNS injectant simultanément :
  * **Task 2 :** Un faux enregistrement `A` associant `www.example.net` à `10.0.2.5`.
  * **Task 3 :** Deux faux serveurs autoritaires `NS` (`ns1.example.net` et `ns2.example.net`) pour toute la zone `example.net`.
  * **Task 5 :** Des enregistrements additionnels (*Glue records*) associant ces faux serveurs NS aux IP `1.2.3.4` et `5.6.7.8`.
* Exécution de requêtes `dig www.example.net` depuis la machine utilisateur (`user-10.9.0.5`) pour valider la redirection et la mise en cache.

### 2. Ce que j'ai observé
* La première commande `dig www.example.net` renvoie immédiatement la fausse IP **`10.0.2.5`** dans la section *ANSWER*, ainsi que les faux serveurs `NS` et *Glue records* dans les sections *AUTHORITY* et *ADDITIONAL*.
* Une seconde requête `dig` exécutée immédiatement après répond en `0 msec` avec les mêmes fausses informations, prouvant que les enregistrements empoisonnés sont bien stockés dans le cache du serveur DNS local.

### 3. Ce que j'ai compris & Questions de débriefing
* **Confiance par ID de transaction :** Le DNS historique transite en clair sur UDP et repose uniquement sur un identifiant de transaction de 16 bits et un port source. Tout attaquant présent sur le chemin réseau qui voit passer la requête peut forger et envoyer une réponse plus rapidement que le vrai serveur autoritaire.
* **Combien de temps la victime restera-t-elle trompée ?**  
  Pendant toute la durée du **TTL (Time To Live)** inscrite dans la fausse réponse (`259200` secondes, soit **3 jours**), ou jusqu'à une purge manuelle du cache (`rndc flush`) ou un redémarrage du serveur DNS.
* **Pour quels noms ?**  
  Grâce à l'injection des enregistrements `NS` et des *Glue records*, la victime est trompée pour **l'ensemble des sous-domaines de la zone `*.example.net`** (ex: `mail.example.net`, `login.example.net`), car le serveur DNS local interrogera désormais les serveurs de l'attaquant pour toute requête concernant ce domaine.

---

## Exercice 3 : Rendre un service indisponible - TCP SYN Flood

### 1. Ce que j'ai fait
* Vérification que la protection par SYN cookies était bien désactivée (`net.ipv4.tcp_syncookies = 0`) sur la victime (`victim-10.9.0.5`), et ajustement de la taille de file d'attente (`net.core.somaxconn = 8`) pour reproduire la saturation.
* Lancement d'un flood haute vitesse de paquets TCP `SYN` avec des adresses IP sources usurpées aléatoires (`hping3 -S -p 23 --flood --rand-source 10.9.0.5`) vers le service Telnet de la victime.
* Contrôle des connexions semi-ouvertes avec `netstat -na | grep SYN_RECV` et tentative de connexion légitime depuis `user1-10.9.0.6`.
* Activation de la parade (`sysctl -w net.ipv4.tcp_syncookies=1`) pendant que l'attaque était toujours active (Task 1.3).

### 2. Ce que j'ai observé
* **Pendant l'attaque (`tcp_syncookies=0`) :** La table réseau de la victime est saturée de dizaines de connexions en état `SYN_RECV` provenant d'IP aléatoires. La tentative de connexion Telnet de l'utilisateur légitime reste figée sur `Trying 10.9.0.5...` sans jamais aboutir.
* **À l'arrêt de l'attaque ou après activation des SYN cookies (`tcp_syncookies=1`) :** L'utilisateur légitime accède immédiatement à l'invite de connexion (`login:`), même lorsque le flood haute vitesse continue de frapper le port `23`.

### 3. Ce que j'ai compris & Questions de débriefing
* **Pourquoi le labo désactive-t-il exprès une protection sur la victime ?**  
  Les **SYN cookies** sont activés par défaut sur les noyaux Linux modernes. Sans les désactiver, le serveur n'alloue pas de mémoire dans sa file d'attente (*backlog*) lors de la réception d'un paquet `SYN` et l'attaque par épuisement des ressources échoue immédiatement.
* **Fonctionnement des SYN cookies :** Au lieu de réserver de la mémoire pour une connexion semi-ouverte, le serveur encode les paramètres de la session dans le numéro de séquence initial (`ISN`) de sa réponse `SYN-ACK`. Il ne crée l'entrée en mémoire que si le client renvoie un paquet `ACK` valide prouvant que son adresse IP n'est pas usurpée.

---

## Synthèse générale du TP3

### 1. Capacités d'un attaquant selon sa position sur le réseau
* **Hors du réseau local (il ne voit rien passer) :** Il ne peut pas écouter le trafic local ni réaliser d'ARP spoofing, mais il peut toujours mener des attaques distantes par saturation vers des services publics exposés (ex: **TCP SYN Flood**).
* **Écoute passive sur un réseau partagé (il voit passer le trafic) :** Sur un réseau Wi-Fi ouvert ou un segment partagé, il peut sniffer tous les protocoles en clair (Telnet, HTTP, requêtes DNS) et gagner des courses de vitesse en forgeant des réponses plus rapides que les serveurs légitimes (**DNS Cache Poisoning**).
* **Position Man-in-the-Middle actif (il est au milieu) :** Grâce à l'**ARP Spoofing**, il force tout le trafic bidirectionnel à transiter par sa machine. Il peut alors lire, bloquer, injecter ou altérer n'importe quel paquet à la volée.

### 2. Pourquoi ARP, DNS et TCP se laissent tromper
* **ARP :** Protocole historique conçu pour un réseau de confiance ; il est sans état et n'authentifie jamais l'expéditeur d'une association IP/MAC.
* **DNS :** Transite en clair sur UDP et se fie uniquement à un identifiant de transaction de 16 bits visible par n'importe quel observateur sur le chemin.
* **TCP :** Le serveur alloue des ressources mémoire dès la première étape du *3-way handshake* (`SYN`), avant même d'avoir vérifié si l'adresse IP source existe réellement.

### 3. Les défenses et leur point commun
* **ARP :** **Dynamic ARP Inspection (DAI)** sur les commutateurs managés (couplé au *DHCP Snooping*) pour valider les correspondances IP/MAC légitimes.
* **DNS :** **DNSSEC** (signature cryptographique des enregistrements DNS) et chiffrement du transport (**DoH / DoT**).
* **TCP :** **SYN Cookies** (validation cryptographique sans état avant toute allocation de mémoire).
* **Point commun :** Toutes ces contre-mesures remplacent la **confiance aveugle** par une **vérification d'authenticité** (par signature cryptographique ou validation d'état) avant d'accepter une information ou d'allouer une ressource système.
