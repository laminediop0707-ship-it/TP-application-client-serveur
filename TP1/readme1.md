# TP1 : Le modèle client-serveur

**Module :** Applications Client-Serveur et Web (ACSW), DUT2 / Licence 2 Réseaux et Télécommunications
**École :** ESP, Département Génie Informatique, UCAD
**Encadrant :** Dr Keba GUEYE
**Binôme :** Boubacar AW et Mouhamadou Lamine DIOP
**Année universitaire :** 2026-2027

Le rapport complet, avec toutes les captures d'écran numérotées, est dans [`rapport_TP1.pdf`](rapport_TP1.pdf). Ce README contient les réponses aux questions Q1 à Q16 et les sorties de commandes utiles. Les mots de passe des comptes du TP sont ceux de l'énoncé et ne sont pas reproduits ici.

## Contenu du dossier

| Fichier | Rôle |
|---|---|
| `README.md` | Réponses aux questions (ce fichier) |
| `rapport_TP1.pdf` | Compte rendu avec captures |
| `scolarite.sql` | Création de la base `scolarite` (fichier du kit) |
| `utilisateurs_tp1.sql` | Création des comptes d'accès distant (fichier du kit) |
| `index.html` | Page servie par Apache (fichier du kit) |

## Environnement

TP réalisé sous VMware Workstation. Chaque VM a deux cartes : `ens33` (NAT, Internet) et `ens37` (réseau privé hôte VMnet1, 192.168.56.0/24, réseau du TP).

| Machine | Adresse | Rôle |
|---|---|---|
| Hôte Windows | 192.168.56.1 (VMnet1) | Client 1 : navigateur, HeidiSQL |
| VM1 (Ubuntu 24.04) | 192.168.56.11 (`ens37`) | Serveur : MariaDB 10.11.14 (3306), Apache 2.4.58 (80) |
| VM2 (Ubuntu 24.04) | 192.168.56.12 (`ens37`) | Client 2 : `mariadb`, `curl` |

Le pare-feu `ufw` de VM1 est inactif : aucune règle n'a été nécessaire pour les ports 3306 et 80. L'interface de capture est `ens37` (l'énoncé citait `enp0s8`, nom VirtualBox).

---

## Partie 1 : Préparer le serveur

### Q1. Contenu de la base et schéma

La base contient **30 étudiants**, **6 EC** et **180 notes** (30 × 6 : une note par étudiant et par EC).

| Table | Colonnes | Clés |
|---|---|---|
| `filiere` | code, libelle | PK : code |
| `etudiant` | matricule, prenom, nom, filiere, email | PK : matricule ; FK : filiere → filiere.code ; email UNIQUE |
| `ec` | code, intitule, credits | PK : code |
| `note` | id, matricule, ec, valeur | PK : id ; FK : matricule → etudiant.matricule ; FK : ec → ec.code ; UNIQUE (matricule, ec) |

Une filière contient plusieurs étudiants, un étudiant a plusieurs notes, un EC est noté pour plusieurs étudiants. `note` est la table d'association entre `etudiant` et `ec`.

### Q2. Client et serveur : même programme ?

**Non.** `ps -ef | grep -i mariadb` montre deux processus distincts :

```text
mysql   1684     1 0 22:43 ?     00:00:02 /usr/sbin/mariadbd
root    3969  3308 0 23:42 pts/0 00:00:00 sudo mariadb
root    3970  3969 0 23:42 pts/1 00:00:00 sudo mariadb
root    3971  3970 0 23:42 pts/1 00:00:00 mariadb
```

- `mariadbd` (PID 1684, utilisateur `mysql`) est le **serveur**, lancé au démarrage et toujours actif.
- `mariadb` (PID 3971, utilisateur `root`) est le **client**, lancé par `sudo mariadb`, qui n'existe que le temps de la session.

Ils sont sur la même machine mais restent deux programmes aux rôles séparés.

---

## Partie 2 : Ouvrir le service au réseau

### Q3. Erreur 10061

Le code **10061** est le code Windows « connexion refusée » (WSAECONNREFUSED). La machine cible a reçu la demande de connexion TCP, mais aucun programme n'écoutait sur cette adresse et ce port, et elle l'a refusée immédiatement.

- **VM1 était joignable** : elle a répondu. Sinon l'erreur aurait été un délai dépassé (10060).
- **MariaDB fonctionnait**, mais il n'écoutait que sur `127.0.0.1:3306`, donc rien n'acceptait les connexions arrivant par `192.168.56.11:3306`.

### Q4. Écouter sur 127.0.0.1 ou 0.0.0.0

- `127.0.0.1` (boucle locale) : le serveur n'accepte que les connexions de sa propre machine.
- `0.0.0.0` : le serveur accepte les connexions sur toutes les interfaces, donc depuis le réseau.

Ubuntu choisit `127.0.0.1` par sécurité : un service fraîchement installé, pas encore configuré ni protégé (comptes, mots de passe, pare-feu), ne doit pas être exposé au réseau. L'administrateur l'ouvre explicitement quand il en a besoin.

Configuration appliquée dans `/etc/mysql/mariadb.conf.d/50-server.cnf` : `bind-address = 0.0.0.0`, puis `sudo systemctl restart mariadb`. Vérification :

```text
$ sudo ss -tlnp | grep 3306
LISTEN 0 80  0.0.0.0:3306  0.0.0.0:*  users:(("mariadbd",pid=4757,fd=29))
```

### Q5. USER() et CURRENT_USER()

Avec le compte `etudiant` depuis Windows :

| USER() | CURRENT_USER() |
|---|---|
| `etudiant@192.168.56.1` | `etudiant@192.168.56.%` |

`USER()` est l'identité **présentée par le client** (nom saisi et adresse réelle d'origine). `CURRENT_USER()` est le **compte retenu par le serveur** pour les droits. Un compte MariaDB est le couple (utilisateur, machine d'origine) : l'adresse 192.168.56.1 correspond au compte défini avec le joker `%`.

### Q6. Connexion root depuis le réseau

Aucun compte root n'existe pour l'accès distant : `root` n'est défini que pour `localhost` et s'authentifie par socket Unix. Une tentative `mariadb -h 192.168.56.11 -u root -p` est donc **refusée** par le serveur (erreur d'accès refusé, de type 1130 ou 1045). C'est une bonne chose : le compte le plus puissant n'est pas exposé au réseau, ce qui limite les conséquences d'une attaque ou d'une fuite de mot de passe.

---

## Partie 3 : Plusieurs clients, un seul serveur

### Q7. Connexions observées côté serveur

`PROCESSLIST` exécuté sur VM1 pendant que Windows (`etudiant`) et VM2 (`secretariat`) sont connectés :

```text
| ID | USER        | HOST                | DB        | COMMAND | TIME | INFO               |
| 35 | root        | localhost           | NULL      | Query   |    0 | SELECT ID, USER... |
| 34 | secretariat | 192.168.56.12:45152 | scolarite | Sleep   |  352 | NULL               |
| 32 | etudiant    | 192.168.56.1:59656  | scolarite | Sleep   |  786 | NULL               |
```

| Client | IP | Port source |
|---|---|---|
| VM2 (`secretariat`) | 192.168.56.12 | 45152 |
| Windows (`etudiant`) | 192.168.56.1 | 59656 |

Ces ports sont des **ports éphémères choisis par le système d'exploitation de chaque client** (Linux pour VM2, Windows pour l'hôte), pas par le serveur. Ils diffèrent de 3306, qui est le port d'écoute connu du serveur. Une connexion TCP est identifiée par (IP client, port client, IP serveur, port serveur) : avec des ports sources différents, le serveur distingue les deux clients même s'ils visent le même port 3306. La ligne ID 35 est la session d'observation (`root`, locale).

### Droits des comptes

Privilèges définis dans `utilisateurs_tp1.sql` : `etudiant` en lecture seule (`SELECT` sur `scolarite.*`) ; `secretariat` en lecture sur toute la base, avec `INSERT` et `UPDATE` sur la seule table `note` (pas de `DELETE`). Le serveur traite donc les commandes de l'énoncé ainsi :

| Où / compte | Commande | Réponse du serveur (d'après les privilèges) |
|---|---|---|
| Windows / etudiant | `UPDATE note SET valeur = 20 …` | Refusé : ERROR 1142, UPDATE command denied |
| VM2 / secretariat | `UPDATE note SET valeur = 12 …` | Accepté : 1 ligne modifiée |
| VM2 / secretariat | `DELETE FROM note WHERE matricule = 'ESP260001'` | Refusé : ERROR 1142, DELETE command denied |
| Windows / etudiant | `SELECT valeur FROM note …` | Renvoie 12.00 |

### Q8. Où sont les droits, qui les vérifie ?

Les droits sont enregistrés **côté serveur**, dans les tables système de la base `mysql`, et c'est le **serveur** qui les vérifie à chaque requête. Le client ne décide de rien : il envoie sa demande et le serveur répond ou refuse (erreur 1142). Conclusion : la sécurité ne peut pas reposer sur le programme client, qu'un utilisateur peut modifier, remplacer ou contourner. Le serveur est le seul point de contrôle fiable, d'où l'intérêt de comptes distincts avec le minimum de droits (principe du moindre privilège).

### Q9. Cohérence des données

Oui, Windows voit **immédiatement** la note modifiée par VM2 : il n'existe qu'**une seule copie** des données, sur le serveur, et aucun client n'en garde de copie locale. Chaque lecture interroge la même source, donc renvoie la valeur à jour, quel que soit le client. Avec des copies dispersées chez les clients, il faudrait les synchroniser et on risquerait d'avoir des versions différentes.

---

## Partie 4 : Le serveur web

Apache (`apache2`) est actif sur VM1 et écoute sur `*:80`. `index.html` est copié dans `/var/www/html/`. Le navigateur Windows affiche la page « Service scolarité » à `http://192.168.56.11/`.

### Q10. Requête et réponse dans `curl -v`

Sortie de `curl -v http://192.168.56.11/` (VM2), début :

```text
* Connected to 192.168.56.11 (192.168.56.11) port 80
> GET / HTTP/1.1
> Host: 192.168.56.11
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Thu, 19 Nov 2026 02:55:19 GMT
< Server: Apache/2.4.58 (Ubuntu)
< Last-Modified: Thu, 19 Nov 2026 02:45:28 GMT
< ETag: "479-660981ece42e0"
< Accept-Ranges: bytes
< Content-Length: 1145
< Vary: Accept-Encoding
< Content-Type: text/html
<
<!doctype html>
...
```

Les lignes `>` sont la **requête** du client, les lignes `<` sont la **réponse** du serveur.

| | Requête (`>`) | Réponse (`<`) |
|---|---|---|
| Première ligne | `GET / HTTP/1.1` : méthode, ressource, version | `HTTP/1.1 200 OK` : version, code d'état, message |
| En-têtes | `Host`, `User-Agent`, `Accept` | `Date`, `Server`, `Last-Modified`, `ETag`, `Content-Length`, `Content-Type` |
| Corps | aucun (GET) | la page HTML, après une ligne vide |

### Q11. Informations enregistrées dans `access.log`

```text
192.168.56.1  - - [19/Nov/2026:02:48:42 +0000] "GET / HTTP/1.1" 200 1000 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/157.0.0.0 Safari/537.36"
192.168.56.1  - - [19/Nov/2026:02:48:42 +0000] "GET /favicon.ico HTTP/1.1" 404 491 "http://192.168.56.11/" "Mozilla/5.0 ..."
192.168.56.1  - - [19/Nov/2026:02:49:34 +0000] "-" 408 0 "-" "-"
192.168.56.12 - - [19/Nov/2026:02:55:19 +0000] "GET / HTTP/1.1" 200 1398 "-" "curl/8.5.0"
```

Pour chaque requête, le serveur note : l'**IP du client**, la **date et l'heure**, la **requête** (méthode, ressource, version), le **code d'état**, la **taille** de la réponse et le **User-Agent**.

- **192.168.56.1 (Windows)** : `GET /` en 200 avec un User-Agent de navigateur (Chrome). On voit aussi `GET /favicon.ico` (404, l'icône n'existe pas) et une ligne 408 : connexion ouverte par le navigateur sans requête envoyée.
- **192.168.56.12 (VM2)** : `GET /` en 200 avec le User-Agent `curl/8.5.0`.

### Q12. Remise d'un segment TCP au bon programme

VM1 héberge deux serveurs sur la même adresse IP. Le système regarde le **port de destination** du segment : 80 pour Apache, 3306 pour MariaDB. Chaque serveur s'est enregistré auprès du système sur son port en se mettant à l'écoute (`ss -tlnp`). Le client doit connaître **l'adresse ET le port** à l'avance, car c'est lui qui prend l'initiative : l'adresse désigne la bonne machine, le port désigne le bon service sur cette machine. Le serveur ne peut pas rappeler un client qu'il ne connaît pas.

### Q13. Connexion persistante et requêtes indépendantes

| | Avantage | Inconvénient |
|---|---|---|
| **Connexion persistante (MariaDB)** | L'état est conservé (utilisateur, base courante, transactions) ; pas de reconnexion à chaque requête | Chaque client occupe des ressources du serveur pendant toute la session, même inactif |
| **Requêtes indépendantes (HTTP)** | Le serveur ne garde rien entre deux requêtes : il supporte beaucoup de clients et se répartit facilement | Le serveur ne reconnaît pas l'utilisateur d'une requête à l'autre |

Un site web « se souvient » d'un utilisateur grâce aux **cookies et sessions** : après l'authentification, le serveur crée une session et envoie un identifiant au navigateur (cookie), qui le renvoie à chaque requête. Le serveur retrouve ainsi l'état associé.

---

## Partie 5 : Les limites du modèle

### Q14. Arrêt du serveur : messages et défaut d'architecture

`sudo systemctl stop mariadb` passe le service à `inactive (dead)`. Une connexion depuis VM2 donne :

```text
$ mariadb -h 192.168.56.11 -u secretariat -p scolarite
ERROR 2002 (HY000): Can't connect to server on '192.168.56.11' (115)
```

Sous Windows, le même refus s'affiche avec le code 10061. **Tous les clients sont affectés** en même temps : ni Windows ni VM2 ne peuvent lire ou écrire des données, alors que leurs machines fonctionnent. Ce défaut s'appelle un **point unique de défaillance** (SPOF, *Single Point of Failure*) : toutes les données et tous les accès dépendent d'une seule machine. On le réduit avec la réplication, des serveurs redondants en haute disponibilité et des sauvegardes. Apache, service distinct, n'est pas affecté par l'arrêt de MariaDB. Le serveur est ensuite redémarré (`sudo systemctl start mariadb`) et l'écoute sur 3306 est rétablie.

### Q15. Contenu de la capture et protection des échanges

Capture sur VM1 : `sudo tcpdump -i ens37 -A -s0 'tcp port 3306'`, pendant qu'un client Windows se connecte puis exécute `SELECT nom, valeur FROM note JOIN etudiant USING(matricule) WHERE matricule = 'ESP260011';`.

Le protocole MariaDB sans chiffrement transporte les échanges **en clair**. Une capture avec `-A` montre lisibles :

- le **nom du compte** (`etudiant`) dans le paquet d'ouverture de connexion ;
- le **texte de la requête** SQL ;
- les **données de la réponse** (nom de l'étudiant, valeurs des notes).

Le **mot de passe n'apparaît pas en clair** : MariaDB utilise un échange défi-réponse (le serveur envoie une valeur aléatoire, le client répond par un condensat calculé à partir du mot de passe). Tout le reste reste lisible par quiconque peut écouter le réseau. Solutions pour protéger les échanges :

- activer **TLS/SSL** sur MariaDB (certificat et clé côté serveur, `REQUIRE SSL` sur les comptes ; HeidiSQL a un onglet SSL) ;
- passer par un **tunnel SSH** (HeidiSQL a un onglet « Tunnel SSH ») ;
- faire transiter le trafic dans un **VPN**.

---

## Synthèse

| Caractéristique | MariaDB | Apache (HTTP) |
|---|---|---|
| Machine et port du serveur | VM1 (192.168.56.11), port 3306 | VM1 (192.168.56.11), port 80 |
| Logiciel client utilisé | HeidiSQL (Windows), `mariadb` (VM2) | Navigateur (Windows), `curl` (VM2) |
| Qui prend l'initiative de la connexion ? | Le client | Le client |
| Durée de vie d'une connexion | Toute la session (persistante) | Une requête et sa réponse (sans état) |
| Où sont vérifiés les droits ? | Sur le serveur (comptes utilisateur@machine et privilèges) | Sur le serveur (configuration d'Apache) |
| Effet d'un arrêt du serveur | Plus aucun accès aux données, tous les clients touchés | Plus aucune page servie, tous les clients touchés |

### Q16. Définition du modèle client-serveur

Le modèle client-serveur est une architecture dans laquelle un programme, le **serveur**, se met à l'écoute sur un **port connu** d'une **machine connue** et attend des demandes. D'autres programmes, les **clients**, prennent toujours l'initiative de se connecter, envoient une requête, puis attendent la réponse. Nos observations le confirment : sur VM1, MariaDB (3306) et Apache (80) attendent, tandis que Windows et VM2 se connectent avec des ports sources choisis par leur propre système. VM1 distingue les services par le port de destination, et les clients par leur couple adresse/port. Les clients ne communiquent jamais entre eux : tout passe par le serveur, qui centralise les données et décide des droits. La modification faite depuis VM2 est visible immédiatement depuis Windows, et les refus d'écriture viennent du serveur, pas du client.

**Avantages**

- **Centralisation et cohérence des données** : une seule copie, donc tous les clients voient la même information à jour (Q9).
- **Contrôle des droits géré au même endroit** : le serveur vérifie comptes et privilèges, quel que soit le client (Q8).

**Inconvénients**

- **Point unique de défaillance** : si le serveur s'arrête, tous les clients perdent le service (Q14).
- **Trafic en clair** : sans TLS, tunnel SSH ou VPN, requêtes et données peuvent être lues sur le réseau (Q15).

---

## Rendu Git

Le compte rendu est rendu sous forme de tag Git :

```bash
git add TP1
git commit -m "TP1 : modèle client-serveur"
git tag TP1
git push origin main
git push origin TP1
```
