# Compte-rendu - TP1 OSINT & Stéganographie

---

## Mission 0 - Lancement de la cible

L'environnement de test a été déployé avec succès sur la machine Kali Linux à l'aide de Docker Compose. Le conteneur de l'application cible tourne correctement.
* **Point de contrôle :** L'application est accessible localement à l'adresse `http://localhost:8080`.
* **Statut du conteneur :** Actif (`Container tp1-cible-acme Started` sur le port `8080`).

---

## Mission 1 - La Vitrine ACME-BE

### 1. Organigramme et informations de contact du site public
En explorant les pages publiques de la vitrine, les profils des employés officiels ont été identifiés :
* **Jean Lefevre** : CEO (Direction générale)
* **Sophie Maes** : Responsable IT (Gestion de l'infrastructure et sécurité)
* **Karim Haddad** : Développeur (Chargé des projets web)

**Format standard des adresses e-mail :** `p.nom@acme-be.example` (première lettre du prénom, point, nom de famille).
* *Note : Grâce à cette convention, l'adresse e-mail probable du CEO est j.lefevre@acme-be.example.*

### 2. Flags trouvés et fuites d'informations
* 🚩 **Flag 1 (Dossier secret) :** `ACME{robots_cache_le_backup}`  
  Trouvé en analysant le fichier public `http://localhost:8080/robots.txt` qui exposait par erreur une directive `Disallow: /interne-old/` menant vers l'ancienne page de sauvegarde de l'entreprise.
* 🚩 **Flag 2 (Sécurité physique) :** `ACME{zoom_sur_le_post_it}`  
  Découvert en inspectant visuellement la photo de l'équipe (`/files/equipe-2024.jpg`). Un post-it collé sur le tableau blanc en arrière-plan divulguait le mot de passe du réseau sans fil.

### 3. Réflexion : Publication volontaire vs Fuite
La page équipe est publiée volontairement pour des besoins professionnels. Le dossier ancien et le mot de passe Wi-Fi sur le post-it sont des fuites involontaires. Cette différence est critique car les fuites offrent des accès directs à un attaquant sans nécessiter d'exploitation complexe.

---

## Mission 2 - Les fichiers parlent (Métadonnées)

### 1. Extraction des données (Flags 3, 4, 5, 9)
L'utilisation de l'outil `exiftool` sur les différents fichiers du blog a permis de récolter les éléments suivants :
* 🚩 **Flag 3 (Adresse du siège d'ACME) :** Place de l'Université, 1348 Ottignies-Louvain-la-Neuve, Belgique.  
  *Détails :* Coordonnées géographiques brutes extraites de la photo du siège (`50°40'09.84"N, 4°36'51.84"E`) puis décodées via Google Maps.
* 🚩 **Flag 4 (Texte caché dans le PDF) :** `ACME{pdf_metadata_auteur}`  
  Extrait directement du champ *Keywords* (mots-clés) du document `rapport-2025.pdf`.
* 🚩 **Flag 5 (Outil de création) :** `Microsoft Word 2019`  
  Identifié dans la ligne *Producer* des métadonnées du document `rapport_public.pdf` (généré par l'utilisateur `j.lefevre`).
* 🚩 **Flag 9 (QR Code Wi-Fi) :** `ACME{qr_code_scanne}`  
  Obtenu en analysant l'image du QR code de la section Wi-Fi à l'aide de l'outil `zbarimg`. La chaîne brute décodée contient la configuration d'accès : `WIFI:S:ACME-Guest;T:WPA;P:ACME{qr_code_scanne};;`.

### 2. Réponses aux questions de réflexion
* **L'employé masqué de l'organigramme :** Un quatrième utilisateur portant les initiales **MV** a été découvert dans les fichiers système (`notes.txt`). En recoupant avec la page `/interne-old/`, il s'agit de **M. Vandenberghe, administrateur système**, qui n'apparaît nulle part sur la vitrine publique.
* **Pourquoi une photo de téléphone les coordonnées GPS ?** Les smartphones récents utilisent le module GPS intégré pour ajouter automatiquement des balises "EXIF" (métadonnées) aux images. Cela permet de géolocaliser précisément le lieu de la prise de vue si l'utilisateur n'a pas désactivé cette option de confidentialité.
* **Qu'est-ce qu'un fraudeur peut faire d'un nom de compte Windows ?** Un attaquant peut utiliser des structures d'identifiants (comme `j.lefevre` ou `s.maes`) pour mener des attaques ciblées d'ingénierie sociale (phishing) ou pour exécuter des attaques de force brute / credential stuffing sur les portails d'accès externes de l'entreprise (comme leur VPN en cours de migration).

### 3. Profils de la cible & Scénarios de Phishing
* **Jean Lefevre (CEO) :** Grand utilisateur de Microsoft Word 2019. Absent du 17 au 20 novembre 2026 pour le salon EuroElec à Munich.
  * *Scénario :* Un e-mail urgent simulant une alerte de licence Word expirée juste avant son départ à Munich, ou une fausse relance de l'organisation EuroElec demandant une validation de document.
* **Sophie Maes (IT) :** En charge de l'infrastructure et de la migration critique du VPN.
  * *Scénario :* Une fausse notification de support technique signalant une erreur de synchronisation majeure sur le nouveau serveur VPN.
* **Karim Haddad (Développeur) :** Travaille activement sur le projet secret de nouveau site e-commerce de l'entreprise.
  * *Scénario :* Un faux rapport de bug ou une alerte de sécurité GitHub concernant une dépendance logicielle corrompue dans le code du site e-commerce.

---

## Mission 3 - Les fichiers qui mentent (Stéganographie)

### 1. Résolution des défis (Flags 6, 7, 8)
* 🚩 **Flag 6 (L'image double) :** `ACME{binwalk_zip_planque}`  
  Découvert en analysant le fichier `logo.png` avec l'outil `binwalk`. L'image dissimulait une archive ZIP cachée contenant un fichier secret `secret.txt` qui a été extrait via la commande `binwalk -e`.
* 🚩 **Flag 7 (La chaîne faussement chiffrée) :** `ACME{base64_facile_a_lire}`  
  Trouvé dans le fichier de configuration `notes.txt`. La chaîne n'était pas chiffrée mais simplement encodée. Elle a été lue à l'aide de la commande `echo "QUNNRXtiYXNlNjRfZmFjaWxlX2FFbGlyZX0=" | base64 -d`.
* 🚩 **Flag 8 (Le carré noir et blanc) :** `ACME{qr_code_scanne}`  
  *(Correspond également au Flag 9 de la mission précédente).* Représente le décodage réussi du QR Code de la section Wi-Fi invités via l'utilitaire `zbarimg`.

### 2. Réponses aux questions théoriques
* **Quelle différence entre encoder et chiffrer ?**
  * *Encoder* (ex: Base64) sert uniquement à changer le format d'affichage d'une donnée pour qu'elle soit compatible avec différents protocoles informatiques. Tout le monde peut la décoder librement, il n'y a aucune sécurité.
  * *Chiffrer* (ex: AES) consiste à rendre une information totalement incompréhensible à l'aide d'une clé de sécurité. Seules les personnes possédant la clé de déchiffrement secrète peuvent retrouver la donnée originale.
* **Pourquoi ne jamais se fier à l'extension d'un fichier ?**
  * L'extension (`.png`, `.txt`, `.pdf`) n'est qu'une étiquette visuelle modifiable à volonté. Un attaquant peut renommer un fichier exécutable malveillant en `.png` pour tromper la victime. Pour connaître la nature réelle d'un document, il faut lire sa signature brute (les *magic bytes*) au début du fichier avec une commande système comme `file`.
