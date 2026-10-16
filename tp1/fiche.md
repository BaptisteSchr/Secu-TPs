# Compte-rendu - TP1 OSINT

## Mission 0 - Lancement de la cible
La cible Docker ACME a été déployée et activée avec succès sur le port 8080.

<img width="651" height="199" alt="Capture d&#39;écran 2026-10-16 150950" src="https://github.com/user-attachments/assets/7684a894-b517-4ea0-b557-72fd9ef13c03" />

<img width="641" height="504" alt="Capture d&#39;écran 2026-10-16 152308" src="https://github.com/user-attachments/assets/a6d18914-b323-4c7c-9ae5-154a8ccff784" />



## Mission 1 - La Vitrine ACME-BE

### 1. Organigramme et Contacts
* **Jean Lefevre** : CEO (Direction)
* **Sophie Maes** : Responsable IT (Gestion technique / Factures)
* **Karim Haddad** : Développeur

**Format des adresses e-mail :** `p.nom@acme-be.example` (première lettre du prénom, point, nom).
* *Note : L'adresse probable du CEO est j.lefevre@acme-be.example*

<img width="792" height="514" alt="Capture d&#39;écran 2026-10-16 154840" src="https://github.com/user-attachments/assets/c115defa-88db-4e15-82d0-da36ad72b395" />


### 2. Flags trouvés (Fuites d'informations)
* **Flag 1 (robots.txt) :** Découverte d'une directive pointant vers le dossier caché `/interne-old/` avec un commentaire de la responsable IT.
* **Flag 2 (Photo d'équipe) :** Identifiants Wi-Fi d'invités visibles sur un post-it sur le tableau blanc (`/files/equipe-2024.jpg`).
  * **SSID :** ACME-Guest
  * **Mot de passe :** ACME{zoom_sur_le_post_it}

<img width="792" height="168" alt="Capture d&#39;écran 2026-10-16 154808" src="https://github.com/user-attachments/assets/159bf398-d5b7-4401-b3b8-8b735237b613" />

<img width="787" height="489" alt="Capture d&#39;écran 2026-10-16 154920" src="https://github.com/user-attachments/assets/17bb43f4-b319-4a4d-9fca-49bc60b44d41" />



### 3. Réflexion : Publication volontaire vs Fuite
La page équipe est publiée volontairement pour des besoins professionnels. Le dossier ancien et le mot de passe Wi-Fi sur le post-it sont des fuites involontaires. Cette différence est critique car les fuites offrent des accès directs à un attaquant sans nécessiter d'exploitation complexe.

## Mission 2 - Les fichiers parlent

### 1. Informations extraites (Flags 3, 4, 5, 9 + Jeton Base64)
* **Siège d'ACME (Flag 3) :** Place de l'Université, 1348 Ottignies-Louvain-la-Neuve, Belgique (Coordonnées GPS extraites de la photo : `50°40'09.84"N 4°36'51.84"E`).

<img width="643" height="452" alt="Capture d&#39;écran 2026-10-16 161115" src="https://github.com/user-attachments/assets/23db6db7-f075-4030-888d-7b0070c9d08a" />

<img width="438" height="39" alt="Capture d&#39;écran 2026-10-16 161151" src="https://github.com/user-attachments/assets/2eedd457-6eba-45b3-8094-ffab3241ff72" />

<img width="785" height="692" alt="Capture d&#39;écran 2026-10-16 161318" src="https://github.com/user-attachments/assets/7a5bf3ea-dc4c-4bdf-b2f6-f1cdab43114f" />

* **Rapport d'activité 2025 (Flag 4 & 5) :** 
  * Auteur : `j.lefevre` (Jean Lefevre)
  * Outil utilisé : Microsoft Word 2019
  * Date de création : 16 octobre 2026
  * Flag trouvé : `ACME{pdf_metadata_auteur}`

 <img width="637" height="404" alt="Capture d&#39;écran 2026-10-16 161633" src="https://github.com/user-attachments/assets/8e32ea5b-50cd-4e98-a517-82b522c15d66" />


* **Photo de la réunion :** 
  * Auteur : `s.maes` (Sophie Maes)
  * Outil utilisé : ACME PhotoCam 2.0
  * Date : 28 novembre 2024

<img width="653" height="508" alt="Capture d&#39;écran 2026-10-16 164443" src="https://github.com/user-attachments/assets/ee6dfacd-fc46-4381-9528-8279da4f80b3" />

* **QR Code Wi-Fi (Flag 9) :** 
  * Contenu décodé : `WIFI:S:ACME-Guest;T:WPA;P:ACME{qr_code_scanne};;`
  * Flag trouvé : `ACME{qr_code_scanne}`
 
<img width="523" height="100" alt="Capture d&#39;écran 2026-10-16 162822" src="https://github.com/user-attachments/assets/fbf62afc-ede6-4c1c-a933-f66a5a217daf" />

* **Fichier confidentiel notes.txt (Jeton masqué) :**
  * Chaîne décodée : `QUNNRXtiYXNlNjRfZmFjaWxlX2FfbGlyZX0=`
  * Flag trouvé : `ACME{base64_facile_a_lire}`

<img width="798" height="223" alt="Capture d&#39;écran 2026-10-16 162538" src="https://github.com/user-attachments/assets/b5df94e6-a24c-4d88-9128-a7fd5d6fb8d6" />

<img width="536" height="111" alt="Capture d&#39;écran 2026-10-16 163254" src="https://github.com/user-attachments/assets/70a35c49-bfb0-44c1-90e3-36b0e4de4570" />


### 2. Réponses aux questions de réflexion
* **L'employé masqué de l'organigramme :** Une personne portant les initiales **MV** (vraisemblablement un administrateur système ou un développeur externe) a rédigé le fichier `notes.txt` présent dans le répertoire `/files/`, mais n'apparaît pas sur la vitrine officielle de l'équipe.

<img width="798" height="223" alt="Capture d&#39;écran 2026-10-16 162538" src="https://github.com/user-attachments/assets/ef7eb86f-120a-4e30-9a86-590e681f59ef" />

* **Pourquoi une photo de téléphone les coordonnées GPS ?** Les smartphones récents utilisent le module GPS intégré pour ajouter automatiquement des balises "EXIF" (métadonnées) aux images. Cela permet de géolocaliser précisément le lieu de la prise de vue si l'utilisateur n'a pas désactivé cette option de confidentialité.
* **Qu'est-ce qu'un fraudeur peut faire d'un nom de compte Windows ?** Un attaquant peut utiliser des structures d'identifiants (comme `j.lefevre` ou `s.maes`) pour mener des attaques ciblées d'ingénierie sociale (phishing) ou pour exécuter des attaques de force brute / credential stuffing sur les portails d'accès externes de l'entreprise (comme leur VPN en cours de migration).

### 3. Profils de la cible & Faux e-mails crédibles
* **Jean Lefevre (CEO) :** Utilise Microsoft Word 2019. Absent du 17 au 20 novembre 2026 pour le salon EuroElec à Munich.
  * *Faux e-mail :* Un mail urgent prétextant un problème critique de licence Word 2019 juste avant son départ à Munich, ou une fausse confirmation de l'organisation EuroElec demandant une validation immédiate de document.
 
<img width="772" height="109" alt="Capture d&#39;écran 2026-10-16 165539" src="https://github.com/user-attachments/assets/a5a3c5a5-7cf1-44b7-a1ce-be4a16c9ce31" />


* **Sophie Maes (IT) :** Travaille activement sur la migration du VPN de l'entreprise.
  * *Faux e-mail :* Une alerte de sécurité simulée ou un message de support technique indiquant une erreur de synchronisation majeure sur le nouveau serveur VPN en cours de déploiement.
 
<img width="772" height="109" alt="Capture d&#39;écran 2026-10-16 165539" src="https://github.com/user-attachments/assets/87fee441-8d62-42cf-a4b3-6d9a502f067c" />

<img width="792" height="514" alt="Capture d&#39;écran 2026-10-16 154840" src="https://github.com/user-attachments/assets/bdf14dae-ea6a-4d93-b010-8299e1d3b985" />

* **Karim Haddad (Développeur) :** S'occupe de la création du nouveau site e-commerce de l'entreprise (projet en cours).
  * *Faux e-mail :* Une fausse notification de mise à jour de sécurité pour une dépendance logicielle majeure (ex: npm, GitHub) ou un faux retour client signalant un bug d'affichage critique sur la version de test du panier e-commerce.

<img width="591" height="290" alt="Capture d&#39;écran 2026-10-16 162233" src="https://github.com/user-attachments/assets/db0fba08-0aa1-4869-a6a9-8281ab4f531f" />

<img width="792" height="514" alt="Capture d&#39;écran 2026-10-16 154840" src="https://github.com/user-attachments/assets/68d407bd-a23a-416b-8299-b3eada0c6d20" />
