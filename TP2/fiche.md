# Fiche de Surface d'Attaque - Audit Interne ACME (TP2)

## 1. Cartographie des Hôtes et Services (Objectif 1)

Voici l'inventaire complet des 7 cibles identifiées sur le périmètre autorisé :

### Segment DMZ (172.21.0.0/24)
*   **172.21.0.10** : Port `80/tcp` (open) | Service : `http` | Version : `Apache httpd 2.4.69 ((Unix))`
*   **172.21.0.11** : Port `22/tcp` (open) | Service : `ssh` | Version : `OpenSSH 8.9p1 Ubuntu 3ubuntu0.18`
*   **172.21.0.13** : Port `6379/tcp` (open) | Service : `redis` | Version : `Redis key-value store 7.4.11` *(Découvert via scan ciblé -p6379)*
*   **172.21.0.14** : **Aucun port ouvert trouvé (Hôte discret)** | Répond au Ping (Couche 3) mais n'expose aucun service (Couche 4).

### Segment LAN (172.22.0.0/24)
*   **172.22.0.12** : Port `3306/tcp` (open) | Service : `mysql` | Version : `MySQL 8.0.43`
*   **172.22.0.13** : Ports `139/tcp`, `445/tcp` (open) | Service : `netbios-ssn`, `microsoft-ds` | Version : `Samba smbd 4`
*   **172.22.0.14** : Port `161/udp` (open) | Service : `snmp` | Version : `SNMPv1/v3 server (public)` *(Découvert via scan UDP -sU)*

### Cible Locale d'Extension
*   **127.0.0.1:8080** : Port `8080/tcp` (open) | Service : `http` | Version : `Apache httpd 2.4.49 (Unix)` *(CVE-2021-41773)*

---

## 2. Deux Constats Expliqués (Objectif 2 & 3)

### Constat 1 : Fuite de secrets techniques et données en clair (MySQL / SMB)
*   **Preuve :** La lecture du fichier `staff_roster.txt` sur le partage SMB restreint a révélé une note technique indiquant un oubli de nettoyage. L'interrogation de la table `system_config` sur la base MySQL (`172.22.0.12`) a confirmé la présence en clair des identifiants de sauvegarde : `adminservice` / `BaK-2024-c0ld!`.
*   **Condition d'accès :** Accès initial via un compte utilisateur fourni (`jdoe` sur SMB) combiné à un compte d'administration fourni (`root` sur MySQL).
*   **Risque :** Compromission croisée (*Credential Reuse*). Un attaquant peut réutiliser ce mot de passe administrateur pour usurper le service de sauvegarde nocturne, exfiltrer l'ensemble des données de l'entreprise ou détruire les backups.
*   **Recommandation :** Purger immédiatement les lignes contenant le mot de passe en texte brut de la table `system_config`. Migrer la gestion des secrets vers un coffre-fort sécurisé (ex: HashiCorp Vault) et appliquer une rotation immédiate du mot de passe du compte `adminservice`.

### Constat 2 : Interception de secret partagé trivial (SNMPv2c)
*   **Preuve :** L'extraction de données via `tshark` sur l'interface réseau bridge a permis de capturer et de lire en texte clair la communauté SNMP **`public`** transitant vers la cible `172.22.0.14`. Elle a exposé l'emplacement (`HQ Datacenter - Rack 12`) et le modèle précis de l'équipement Cisco simulé.
*   **Condition d'accès :** Écoute passive du réseau local (*Sniffing*) et utilisation d'une configuration par défaut triviale (`public`).
*   **Risque :** Fuite d'informations d'infrastructure stratégiques permettant la préparation d'une attaque ciblée (recherche de CVE sur la version de l'OS) ou d'un déni de service (DoS) sur le routeur de bordure d'ACME.
*   **Recommandation :** Désactiver SNMPv1/v2c et migrer l'infrastructure vers **SNMPv3** pour imposer une authentification forte et le chiffrement AES des paquets. Restreindre l'accès au port UDP 161 via un pare-feu pour n'autoriser que l'IP du serveur de supervision légitime.

---

## 3. Conclusion et Remédiation Globale

### Que corriger en premier ?
La priorité absolue est la mise à jour du serveur Apache vers une version stable et corrigée (2.4.51 ou supérieure) afin de corriger la **CVE-2021-41773** (Path Traversal/RCE) testée en Mission 3, ainsi que le durcissement du serveur Redis qui accepte les connexions anonymes (Mission 1d). Ces services sont en DMZ, donc directement exposés aux attaquants.

### Comment vérifier ce choix sans casser les usages utiles ?
*   **Pour le serveur Web :** Après la mise à jour, rejouer la commande `curl -sS --path-as-is 'http://127.0.0'`. Le serveur doit répondre par un code de blocage **403 Forbidden** au lieu d'afficher le fichier. On valide le maintien des usages utiles en vérifiant qu'un navigateur standard accède toujours normalement aux pages légitimes du site web (HTTP 200).
*   **Pour Redis :** Activer la directive `requirepass` dans le fichier de configuration. Vérifier la correction en rejouant le script Nmap `nmap -p6379 --script redis-info`. Le serveur doit désormais refuser de délivrer ses informations système sans authentification.
