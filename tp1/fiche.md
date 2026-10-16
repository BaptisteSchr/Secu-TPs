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
