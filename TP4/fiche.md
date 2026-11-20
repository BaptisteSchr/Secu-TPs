# Fiche de Synthèse - TP4 Sécurité Web

## 1. Remise : Mécanisme et Défense

* **Phrase clé (Mécanisme + Défense) :**
  > Une entrée non fiable non échappée permet l'injection et l'exécution de code malveillant (comme une requête SQL modifiée ou une balise `onerror` chargeant `hook.js`), ce qu'on corrige à la racine par des **requêtes paramétrées** (SQLi) et l'**encodage en sortie** selon le contexte (XSS), renforcés par une **CSP** et des cookies **`HttpOnly`, `SameSite`, `Secure`**.

---

## 2. Tableau Récapitulatif des Attaques et Défenses (OWASP Top 10 - A05 / A04)

| Attaque / Faille | Ce qui l'a permise | Qui exécute quoi et où ? | Défense côté développement |
| :--- | :--- | :--- | :--- |
| **SQL Injection (Login & UNION)** | Saisie utilisateur (`email`, `q`) concaténée directement dans la requête SQL. | Le **SGBD (SQLite)** exécute la requête SQL modifiée **côté serveur**. | **Requêtes paramétrées (préparées)** + compte BDD aux **droits minimaux** + erreurs génériques. |
| **Hash faible sans sel (Bonus A04)** | Stockage des mots de passe en **MD5 sans sel** (`admin123` cassé rapidement). | L'attaquant casse le hash hors-ligne par dictionnaire (`john` + `rockyou.txt`). | Utiliser une fonction de hachage **lente et salée** (**bcrypt** ou **Argon2**). |
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
