# TP3 - Attaques réseau : ARP, DNS, TCP, qui croit qui ?

* **Environnement :** Machine virtuelle Kali Linux & Laboratoires Docker SEED (`10.9.0.0/24`)
* **Référentiel MITRE ATT&CK :** 
  * `T1557.002` - ARP Cache Poisoning & `T1040` - Network Sniffing (*Credential Access, Collection*)
  * `T1557` - Adversary-in-the-Middle (*Credential Access, Collection*)
  * `T1499.001` - OS Exhaustion Flood (*Impact*)

---

## Exercice 1 : ARP Cache Poisoning & MITM

### 1. Cartographie initiale et empoisonnement ARP (Task 1)
Relevé des adresses IP et MAC réelles des conteneurs du laboratoire (`ip -br link show eth0`) :
* **Victime A (`A-10.9.0.5`) :** `26:29:14:2a:58:c0`
* **Victime B (`B-10.9.0.6`) :** `72:40:be:4c:f7:7d`
* **Attaquant M (`M-10.9.0.105`) :** `96:99:e9:ae:bb:da`

<img width="796" height="851" alt="1" src="https://github.com/user-attachments/assets/101a8fbc-4569-4499-bf12-16c563529b4d" />

**Observations des trois variantes vers `10.9.0.5` (en usurpant `10.9.0.6`) :**
* **Variante A (`ARP Request` - `op=1`) :** Réussit immédiatement. La table ARP de `A` (`arp -n`) associe l'IP `10.9.0.6` à la MAC de l'attaquant (`96:99:e9:ae:bb:da`), même lorsque le cache initial est vide.
* **Variante B (`ARP Reply` - `op=2`) :** Échoue lorsque le cache de la victime est vidé au préalable (`ip -s -s neigh flush all`).
* **Variante C (`Gratuitous ARP` en broadcast) :** Échoue également sur un cache vide.

<img width="792" height="848" alt="2" src="https://github.com/user-attachments/assets/d39e819a-48a6-4eb9-9e56-cdca0fe8928d" />

<img width="794" height="845" alt="3" src="https://github.com/user-attachments/assets/af6914f7-f194-47b6-90f4-7966694cc39e" />

<img width="798" height="843" alt="4" src="https://github.com/user-attachments/assets/c76b0a51-c0bf-4f1e-8766-2b772207abd6" />


### 2. Interception Man-in-the-Middle sur Telnet (Task 2)
* Mise en place d'un script Python Scapy envoyant toutes les 2 secondes des requêtes ARP forgées aux deux victimes (`A` et `B`) et activation du routage IP (`sysctl -w net.ipv4.ip_forward=1`) sur l'attaquant.
* Capture du trafic sur l'interface `any` avec **Wireshark** pendant qu'un utilisateur sur `A` se connecte en Telnet sur `B` (`telnet 10.9.0.6`).
* L'option *Follow TCP Stream* de Wireshark permet de lire l'intégralité de la session en clair : identifiant **`seed`** et mot de passe **`dees`**.

<img width="1594" height="845" alt="6" src="https://github.com/user-attachments/assets/0c02b0b9-86df-4f1b-8a67-3e6dbb2753d1" />

### 3. Réponses aux questions de débriefing
* **Pourquoi les variantes B et C ont-elles échoué sur un cache vide ?**  
  Par sécurité et optimisation (`arp_accept = 0`), le noyau Linux moderne rejette les réponses ARP non sollicitées si l'adresse IP cible ne figure pas déjà dans sa table de voisinage. En revanche, il enregistre toujours l'émetteur d'une requête ARP légitime (Variante A).
* **Qu'est-ce qui aurait empêché la Task 2 de montrer le mot de passe ?**  
  L'utilisation d'un protocole chiffré de bout en bout comme **SSH** au lieu de Telnet. L'attaque ARP MITM intercepterait toujours les paquets, mais leur contenu serait illisible.

---

## Exercice 2 : DNS Cache Poisoning

### 1. Mécanisme de l'attaque (Tasks 2, 3 et 5)
* Identification de l'interface bridge Docker du labo (`br-d9dafec411b3` sur `10.9.0.1/24`) et lancement du script Python Scapy (`dns_sniff_spoof.py`) écoutant le port UDP `53`.
* Interception de la requête récursive émise par le serveur DNS local (`10.9.0.53`) pour `www.example.net`, récupération de l'**ID de transaction (16 bits)** et injection simultanée d'une réponse forgée :
  * **Task 2 :** Faux enregistrement `A` pointant `www.example.net` vers `10.0.2.5`.
  * **Task 3 :** Faux serveurs autoritaires `NS` (`ns1.example.net` et `ns2.example.net`) pour toute la zone `example.net`.
  * **Task 5 :** Enregistrements additionnels (*Glue records*) associant ces faux NS aux IP `1.2.3.4` et `5.6.7.8`.

<img width="796" height="838" alt="7" src="https://github.com/user-attachments/assets/a6f50f15-e514-415c-b737-c6737245aed6" />

### 2. Vérification du cache et réponses aux questions
* Une seconde commande `dig www.example.net` répond en `0 msec` avec les mêmes fausses entrées, prouvant que le cache du serveur DNS local (`10.9.0.53`) est bien empoisonné.

<img width="794" height="827" alt="8" src="https://github.com/user-attachments/assets/68b6fc50-f7fe-4020-88d9-265cb165ea6a" />

* **Combien de temps la victime restera-t-elle trompée ?**  
  Pendant toute la durée du **TTL** inscrite dans le paquet forgé (`259200` secondes, soit **3 jours**), ou jusqu'à une purge manuelle du cache (`rndc flush`) ou un redémarrage du service DNS.
* **Pour quels noms ?**  
  Grâce à l'injection des enregistrements `NS` et des *Glue records*, la victime est trompée pour **tous les sous-domaines de la zone `*.example.net`** (ex: `mail.example.net`, `login.example.net`), car le serveur local interroge désormais les faux serveurs de l'attaquant pour l'ensemble du domaine.

---

## Exercice 3 : Rendre un service indisponible - TCP SYN Flood

### 1. Saturation de la file d'attente (Task 1.1)
* Vérification que la protection par SYN cookies est désactivée (`net.ipv4.tcp_syncookies = 0`) sur la victime (`victim-10.9.0.5`) et réglage de la file d'attente (`net.core.somaxconn = 8`).
* Inondation du port `23` (Telnet) avec des paquets `SYN` forgés provenant d'IP sources aléatoires (`sudo hping3 -S -p 23 --flood --rand-source 10.9.0.5`).
* Contrôle via `netstat -na | grep SYN_RECV` : des dizaines de connexions semi-ouvertes saturent la victime. Une tentative de connexion légitime depuis `user1-10.9.0.6` reste bloquée indéfiniment sur `Trying 10.9.0.5...`.

<img width="793" height="828" alt="10" src="https://github.com/user-attachments/assets/dd2583af-10df-4e93-a661-873b121ecb79" />

<img width="795" height="830" alt="11" src="https://github.com/user-attachments/assets/9a8da26d-6b80-48c9-aef8-cadf3725b2cd" />

<img width="797" height="848" alt="12" src="https://github.com/user-attachments/assets/a08a3b09-0140-4b90-8820-0d36f107d8f6" />

### 2. Activation de la parade (Task 1.3) et débriefing
* Activation des SYN cookies sur la victime pendant que le flood tourne toujours (`sysctl -w net.ipv4.tcp_syncookies=1`) : l'utilisateur légitime accède immédiatement à l'invite `login:`.

<img width="776" height="826" alt="13" src="https://github.com/user-attachments/assets/8b2a19c5-70b9-4f81-b191-cee86752cfd9" />

* **Pourquoi le labo désactive-t-il exprès une protection sur la victime ?**  
  Les **SYN cookies** sont activés par défaut sur les noyaux Linux modernes. Sans les désactiver, le serveur n'alloue pas de mémoire dans sa file d'attente (*backlog*) à la réception d'un `SYN` (il encode l'état dans le numéro de séquence initial `ISN`), ce qui empêche toute saturation.
