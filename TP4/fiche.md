# Fiche de Synthèse — TP4 Sécurité Web (Remise & Révision Examen)

## 1. Remise Moodle : Preuve, Mécanisme et Défense

* **Preuves jointes :**
  * Requête modifiée dans Burp Repeater (`7.png`) : `GET /rest/products/search?q=apple'))%20UNION%20SELECT%201,%27TP4_BAPTISTE%27,3,4,5,6,7,8,9,10%20--`.
  * Résultat obtenu (`8.png`) : réponse `HTTP/1.1 200 OK` retournant `"name": "TP4_BAPTISTE"` dans le JSON (ainsi que l'extraction des comptes `Users` en `12.png`/`13.png` et la finale BeEF en `21.png`/`22.png`).
* **Phrase d'explication (Mécanisme + Défense) :**
  > L'application concatène directement la saisie utilisateur dans sa requête SQL (ou la réaffiche sans échappement dans le DOM), transformant une donnée non fiable en instruction exécutée par la base de données ou le navigateur ; on corrige la faille à la racine par des **requêtes paramétrées** (SQLi) et l'**encodage en sortie** selon le contexte (XSS), renforcés par une **CSP** et des cookies **`HttpOnly`, `SameSite`, `Secure`**.

---

## 2. Tableau Récapitulatif des Attaques et Défenses (OWASP Top 10 — A05 / A04)

| Attaque / Faille | Ce qui l'a permise | Qui exécute quoi et où ? | Défense côté développement |
| :--- | :--- | :--- | :--- |
| **SQL Injection (Login & UNION)** | Saisie utilisateur (`email`, `q`) concaténée directement dans la requête SQL. | Le **SGBD (SQLite)** exécute la requête SQL modifiée **côté serveur**. | **Requêtes paramétrées (préparées)** + compte BDD aux **droits minimaux** + erreurs génériques. |
| **Hash faible sans sel (Bonus A04)** | Stockage des mots de passe en **MD5 sans sel** (`admin123` cassé en `0,21s`). | L'attaquant casse le hash hors-ligne par dictionnaire (`john` + `rockyou.txt`). | Utiliser une fonction de hachage **lente et salée** (**bcrypt** ou **Argon2**). |
| **XSS (DOM & Stockée)** | Donnée utilisateur réaffichée dans la page sans encodage. | Le **navigateur de la victime** exécute le JavaScript injecté **côté client**. | **Encodage en sortie** selon le contexte (HTML, attribut, JS) + **CSP**. |
| **Prise de main (BeEF)** | XSS stockée permettant de charger un script externe (`hook.js`) chez l'admin. | Le **navigateur de la victime** charge et exécute `hook.js` **côté client**. | Corriger la XSS + **CSP** bloquant les scripts externes + cookies **`HttpOnly`, `SameSite`, `Secure`**. |

---

## 3. Points Clés pour l'Examen

1. **Quand une donnée devient-elle une instruction ?**
   * Dès qu'elle est interprétée par un moteur (parseur SQL côté serveur ou parseur HTML/JS côté navigateur) sans séparation stricte entre code et données (requête préparée ou encodage).
2. **Différence entre DOM/Reflected XSS et Stored XSS :**
   * **DOM / Reflected XSS :** Le payload transite dans la requête/URL ; il faut que la victime ouvre un lien piégé.
   * **Stored XSS :** Le payload est enregistré en base de données ; il s'exécute automatiquement chez **toute personne** consultant la page légitime affectée (ex. `/#/administration`).
3. **Distinction Transport (HTTPS) vs Sécurité de l'Application :**
   * **HTTPS** chiffre et authentifie les échanges réseau entre le client et le serveur (empêchant l'écoute ou la modification en transit).
   * Une fois la réponse reçue et déchiffrée par le navigateur, le code de l'application (et tout script XSS injecté) s'exécute normalement : le cadenas vert ne protège pas contre les failles applicatives (SQLi, XSS).
