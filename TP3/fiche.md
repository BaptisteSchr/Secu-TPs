# Fiche de synthèse - TP3 : Attaques réseau (ARP, DNS, TCP)

## 1. Tableau récapitulatif des attaques et défenses

| Protocole & Attaque | Technique MITRE | Faille exploitée (Confiance) | Parade technique |
| :--- | :--- | :--- | :--- |
| **ARP** (*Cache Poisoning / MITM*) | `T1557.002` / `T1040` | Protocole sans état et sans authentification ; accepte aveuglément les associations IP/MAC. | **Dynamic ARP Inspection (DAI)** sur les switchs + chiffrement de bout en bout (**SSH**). |
| **DNS** (*Cache Poisoning*) | `T1557` | Transport en clair sur UDP basé uniquement sur un ID de transaction de 16 bits visible en local. | **DNSSEC** (signature cryptographique des zones) et transport chiffré (**DoH / DoT**). |
| **TCP** (*SYN Flood*) | `T1499.001` | Allocation de mémoire dès le premier paquet `SYN` avant vérification de l'IP source. | **SYN Cookies** (`net.ipv4.tcp_syncookies=1`) pour valider l'état sans réserver de mémoire. |

---

## 2. Résultats clés obtenus par exercice

* **Exercice 1 (ARP & MITM) :** 
  * L'empoisonnement par requête ARP (`op=1`) fonctionne même sur un cache vide, contrairement aux variantes `ARP Reply` et `Gratuitous ARP` bloquées par Linux (`arp_accept=0`).
  * Le placement en *Man-in-the-Middle* entre `10.9.0.5` et `10.9.0.6` a permis de lire en clair sous Wireshark les identifiants Telnet (`seed` / `dees`).
* **Exercice 2 (DNS Spoofing) :** 
  * Interception de la requête DNS sortante de `10.9.0.53` et injection d'une réponse forgée plus rapide contenant un faux enregistrement `A` (`10.0.2.5`), des faux serveurs `NS` et des *Glue records*.
  * La victime est redirigée pour toute la zone `*.example.net` pendant la durée du TTL (`259200` secondes / 3 jours).
* **Exercice 3 (TCP SYN Flood) :** 
  * Saturation de la file d'attente `SYN_RECV` de `10.9.0.5` avec des IP sources aléatoires, bloquant l'accès Telnet légitime (`Trying 10.9.0.5...`).
  * L'activation des **SYN cookies** rétablit instantanément la disponibilité du service même sous attaque active.

---

## 3. Bilan conceptuel (Questions de fin de TP)

1. **Capacités d'un attaquant selon sa position :**
   * *Hors du réseau local (il ne voit rien) :* Ne peut pas sniffer ni faire d'ARP spoofing, mais peut lancer des attaques par saturation à distance (**SYN Flood**).
   * *Écoute passive sur un réseau partagé (il voit passer) :* Peut lire les protocoles en clair (Telnet, HTTP, DNS) et forger des réponses plus rapides que les serveurs légitimes (**DNS Spoofing**).
   * *Position MITM active (il est au milieu) :* Contrôle l'intégralité du trafic bidirectionnel et peut lire, bloquer ou modifier les paquets à la volée (**ARP Spoofing**).
2. **Pourquoi ces protocoles se laissent tromper :**
   * ARP, DNS et TCP ont été conçus historiquement pour fonctionner sur des réseaux de confiance et privilégient la rapidité de communication sur la vérification d'identité.
3. **Le point commun entre toutes les défenses :**
   * Toutes ces contre-mesures (DAI, DNSSEC, SYN Cookies) remplacent la **confiance aveugle** par une **vérification cryptographique ou d'état** avant d'accepter une donnée réseau ou d'allouer des ressources système.
