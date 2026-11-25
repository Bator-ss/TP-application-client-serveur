# TP 1 — Le modèle client-serveur (ACSW)

**Université Cheikh Anta Diop de Dakar (UCAD)**  
**École Supérieure Polytechnique (ESP)** — Département Génie Informatique  
**Filière :** DUT2-TR / Licence 2 Réseaux et Télécommunications (L2SRT)  
**Module :** Applications Client-Serveur et Web (ACSW) — Année universitaire 2026-2027  
**Enseignant :** Dr Keba GUEYE  

**Binôme :**
* **Djamilatou Bikami Diop**
* **Ahmadou Bator Samb**

---

## Environnement expérimental

* **VM1 (Serveur MariaDB 11.8 & Apache) :** `192.168.81.133`
* **VM2 (Client) :** `192.168.81.134`
* **Hôte Windows (Client) :** `192.168.1.7` (vu par VM1 sous l'adresse `192.168.81.1` via la carte virtuelle VMnet8)
* **Comptes MariaDB :** créés pour le motif d'hôte `192.168.81.%`

---

## Partie 1 — Préparer le serveur

### Q1. Contenu de la base et schéma
La base de données `scolarite` contient **30 étudiants**, **6 EC** (éléments constitutifs) et **180 notes** ($30 \times 6$), répartis sur **2 filières** (`II` et `RT`).

#### Schéma des tables (`PK` = clé primaire, `FK` = clé étrangère)

| Table | Colonnes et contraintes |
| :--- | :--- |
| **`filiere`** | `code` (PK), `libelle` |
| **`etudiant`** | `matricule` (PK), `prenom`, `nom`, `filiere` (FK → `filiere.code`), `email` (UNIQUE) |
| **`ec`** | `code` (PK), `intitule`, `credits` |
| **`note`** | `id` (PK), `matricule` (FK → `etudiant.matricule`), `ec` (FK → `ec.code`), `valeur` ; contrainte `UNIQUE(matricule, ec)` |

### Q2. Client et serveur sont-ils le même programme ?
**Non.** `mariadb` est le **client** (interface interactive en ligne de commande) et `mariadbd` est le **serveur** (démon en arrière-plan). Ce sont deux processus distincts possédant chacun son propre PID (`ps -ef | grep -i mariadb`). Lorsqu'ils s'exécutent sur la même machine, ils communiquent localement via un **socket Unix** (par exemple `/run/mysqld/mysqld.sock`).

---

## Partie 2 — Ouvrir le service au réseau

### Q3. Erreur 10061
Le code `10061` (`WSAECONNREFUSED` sous Windows) signifie **« connexion refusée »**.
* **VM1 était-elle joignable ?** Oui, la machine a répondu activement par un segment TCP `RST` (Reset).
* **Le serveur MariaDB fonctionnait-il ?** Oui, le service tournait, mais il n'écoutait que sur l'adresse de boucle locale (`127.0.0.1:3306`) ; aucun processus n'était en écoute sur le port `3306` de l'interface réseau.

### Q4. Écouter sur `127.0.0.1` ou sur `0.0.0.0`
* **`127.0.0.1` (localhost) :** n'accepte que les connexions locales provenant de la machine elle-même.
* **`0.0.0.0` :** écoute sur toutes les adresses IPv4 de toutes les interfaces réseau de la machine, rendant le service accessible depuis le réseau.

Ubuntu configure `127.0.0.1` par défaut afin de **limiter la surface d'attaque** : un service réseau (notamment une base de données) ne doit pas être exposé sans une décision explicite de l'administrateur.

### Q5. `USER()` et `CURRENT_USER()`
Résultat obtenu :
* `USER()` = `etudiant@192.168.81.1`
* `CURRENT_USER()` = `etudiant@192.168.81.%`

**Différence :**
* `USER()` représente l'**identité présentée par le client** telle qu'elle est vue par le serveur (nom d'utilisateur et adresse IP source réelle du client).
* `CURRENT_USER()` représente le **compte réellement retenu par le serveur** (couple `(utilisateur, motif_hote)`) après correspondance avec la table des comptes, et dont les privilèges s'appliquent à la session.

### Q6. Connexion en `root` depuis le réseau
Il n'existe pas de compte `root` distant configuré pour l'accès réseau : le compte administrateur local est `'root'@'localhost'` (authentifié via `unix_socket`). Une tentative de connexion distante depuis Windows échoue donc (erreur `1045 Access denied` ou `1130 Host ... is not allowed`).

C'est une **bonne pratique de sécurité** : le compte d'administration privilégie ainsi l'accès local sécurisé et n'est pas exposé aux tentatives d'intrusion ou d'attaque par force brute à travers le réseau. Les comptes distants doivent rester limités aux privilèges strictement nécessaires.

---

## Partie 3 — Plusieurs clients, un seul serveur

### Q7. `PROCESSLIST` et `ss`
* `information_schema.PROCESSLIST` affiche les sessions actives sur le serveur (par exemple `etudiant` depuis `192.168.81.1:<port_source>` et `secretariat` depuis `192.168.81.134:<port_source>`).
* `ss -tn state established '( sport = :3306 )'` montre les connexions TCP établies vers `192.168.81.133:3306`.

Les **ports sources** sont attribués dynamiquement par le système d'exploitation de chaque client dans sa plage de **ports éphémères**. Ils diffèrent de `3306` (port d'écoute du serveur) et entre eux, car une connexion TCP est unifiée et distinguée par le quadruplet `(IP_source, port_source, IP_destination, port_destination)`.

### Q8. Droits d'accès
* **`etudiant` :** `UPDATE` refusé → `ERROR 1142 (42000): UPDATE command denied to user 'etudiant'@'192.168.81.1' for table 'note'`.
* **`secretariat` :** `UPDATE` autorisé (`1 row affected`) ; `DELETE` refusé → `ERROR 1142 (42000): DELETE command denied to user 'secretariat'@'192.168.81.134' for table 'scolarite'.'note'`.

Les droits sont stockés **sur le serveur** (dans les tables système de la base `mysql`) et **vérifiés par le serveur à chaque requête**, jamais par le client.  
**Conclusion sécurité :** on ne peut jamais faire confiance au client ; changer ou modifier le logiciel client ne permet pas de contourner les contrôles d'accès centralisés côté serveur.

### Q9. Cohérence des données
**Oui**, le client Windows voit immédiatement la modification effectuée depuis VM2 (par exemple la valeur `12.00` remplaçant `9.25`, ensuite rétablie).

La centralisation des données sur un serveur unique garantit cette cohérence car **il n'existe qu'une seule source de vérité** : chaque lecture ou écriture s'exécute directement sur la base centralisée, sans répliques locales désynchronisées chez les clients, et le serveur gère l'ordre/la sérialisation des accès concurrents.

---

## Partie 4 — Le serveur web

### Q10. Sortie de `curl -v`
* **Requête du client (lignes préfixées par `>`) :**
  * **Première ligne :** méthode HTTP (`GET`), ressource demandée (`/`) et version du protocole (`HTTP/1.1`).
  * **En-têtes :** métadonnées envoyées par le client (`Host`, `User-Agent`, `Accept`, etc.).
* **Réponse du serveur (lignes préfixées par `<`) :**
  * **Première ligne :** version HTTP (`HTTP/1.1`), code de statut (`200`) et libellé (`OK`).
  * **En-têtes :** métadonnées renvoyées par le serveur (`Server: Apache/...`, `Content-Type: text/html`, `Content-Length`, etc.).

Une ligne vide sépare les en-têtes du corps de la réponse (le contenu HTML).

### Q11. Informations enregistrées dans `access.log`
Chaque entrée de `/var/log/apache2/access.log` contient notamment :
* L'**adresse IP du client** (`192.168.81.1` pour l'hôte Windows, `192.168.81.134` pour VM2) ;
* La **date et l'heure** de la requête ;
* La **ligne de requête** (`GET / HTTP/1.1`) ;
* Le **code de statut HTTP** (`200`) et la **taille** de la réponse en octets ;
* L'**identifiant `User-Agent`** (navigateur web côté Windows, `curl/...` côté VM2).

### Q12. Plusieurs serveurs sur VM1
* **Identification du processus :** l'adresse IP identifie la machine hôte, tandis que le **port de destination** du segment TCP identifie le service/processus destinataire. Le noyau du système d'exploitation remet le segment au processus ayant ouvert le socket en écoute sur ce port (Apache sur le port `80`, MariaDB sur le port `3306`).
* **Connaissance préalable de l'adresse et du port :** dans le modèle client-serveur, le serveur attend passivement et ne peut pas initier la connexion vers un client quelconque ; c'est le client qui prend l'initiative. Il doit donc connaître l'adresse IP et le port (« ports bien connus » comme `80` ou `3306`) pour joindre le bon service.

### Q13. Connexion persistante ou requête-réponse
* **Connexion persistante (MariaDB) :**
  * *Avantage :* authentification effectuée une seule fois, conservation de l'état de session (transactions, variables de session), absence de reconnexion à chaque requête.
  * *Inconvénient :* le serveur maintient des ressources (mémoire, descripteurs, threads/processus) pour chaque client connecté, ce qui limite le nombre de clients simultanés.
* **Modèle requête-réponse / sans état (HTTP) :**
  * *Avantage :* architecture plus légère, simple et plus facile à faire passer à l'échelle (scalabilité).
  * *Inconvénient :* absence de mémoire automatique d'une requête à l'autre, obligeant à retransmettre le contexte nécessaire.

Un site web « se souvient » d'un utilisateur connecté grâce aux **cookies de session** (un identifiant de session généré après authentification et renvoyé par le navigateur dans l'en-tête `Cookie` à chaque requête) ou à des **jetons** (comme un JWT).

---

## Partie 5 — Les limites du modèle

### Q14. Arrêt du serveur
Lorsque MariaDB est arrêté sur VM1 (`sudo systemctl stop mariadb`) :
* **Client Windows :** `ERROR 2002 (HY000): Can't connect to server on '192.168.81.133' (10061)`.
* **Client VM2 (Linux) :** `ERROR 2002 (HY000): Can't connect to server on '192.168.81.133'` (erreur de connexion refusée / inaccessible, code `111` ou `115`).

**Tous les clients (2 sur 2)** sont impactés. Ce défaut est un **point unique de défaillance** (*Single Point of Failure* — **SPOF**). On peut l'atténuer par la réplication, la haute disponibilité / basculement automatique (*failover*) ou un cluster.

### Q15. Capture du trafic
Une capture réseau (`tcpdump` sur l'interface de VM1 ou Wireshark sur le filtre `mysql`) montre que les échanges MariaDB non chiffrés circulent **en clair** : le nom d'utilisateur (`etudiant`), le texte de la requête SQL (`SELECT ...`) et les données retournées (noms, notes) sont lisibles sur le réseau.

Le **mot de passe n'apparaît pas en clair** (il est transmis sous forme de réponse calculée à partir d'un défi/sel d'authentification), mais les données applicatives restent exposées.

**Solutions pour protéger les échanges :** activer **TLS/SSL** côté MariaDB (`ssl-cert`, `ssl-key`, `require_secure_transport=ON`) ou faire transiter le trafic dans un **tunnel SSH** ou un **VPN**.

---

## Synthèse

| Caractéristique | MariaDB | Apache (HTTP) |
| :--- | :--- | :--- |
| **Machine et port du serveur** | VM1 (`192.168.81.133`), port `3306` | VM1 (`192.168.81.133`), port `80` |
| **Logiciel client utilisé** | HeidiSQL (Windows), `mariadb` (VM2) | Navigateur web (Windows), `curl` (VM2) |
| **Initiative de la connexion** | Le client | Le client |
| **Durée de vie d'une connexion** | Toute la session (persistante) | Une requête-réponse (sans état) |
| **Où sont vérifiés les droits ?** | Sur le serveur (comptes et privilèges) | Sur le serveur (configuration Apache) |
| **Effet d'un arrêt du serveur** | Connexion refusée pour tous les clients | Connexion refusée pour tous les clients |

### Q16. Définition du modèle client-serveur
Le modèle client-serveur est une architecture réseau dans laquelle un programme serveur attend des connexions sur une adresse et un port connus, centralise les données et contrôle les règles d'accès. Les programmes clients prennent l'initiative de la connexion, envoient des requêtes et traitent les réponses, sans communiquer directement entre eux. Dans ce TP, MariaDB (`3306`) et Apache (`80`) s'exécutent sur VM1 tandis que Windows et VM2 s'y connectent depuis des ports éphémères avec des protocoles différents (connexion persistante pour MariaDB, requête-réponse pour HTTP) tout en partageant la même source de données et les mêmes contrôles d'accès côté serveur.

* **Avantages :**
  1. **Centralisation des données et de la sécurité :** cohérence immédiate des informations et vérification des droits imposée par le serveur quel que soit le client.
  2. **Support de clients multiples et variés :** un même serveur peut traiter simultanément plusieurs clients différents (navigateur, `curl`, client SQL, etc.).
* **Inconvénients :**
  1. **Point unique de défaillance (SPOF) :** l'indisponibilité du serveur bloque l'accès de tous les clients (et le serveur peut constituer un goulot d'étranglement sous forte charge).
  2. **Exposition du trafic en clair sans chiffrement :** sans mécanisme de sécurisation (TLS, VPN, tunnel SSH), les requêtes et réponses échangées sur le réseau peuvent être interceptées.
