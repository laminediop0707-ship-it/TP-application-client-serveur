# TP 2 — Architectures 2-tiers et 3-tiers comparées

**Université Cheikh Anta Diop de Dakar — École Supérieure Polytechnique (ESP) — Département Génie Informatique**
Module : Applications Client-Serveur et Web (ACSW) — DUT2 / Licence 2 Réseaux et Télécommunications
Année universitaire 2026-2027

**Réalisé par :** Boubacar AW et Mouhamadou Lamine DIOP
**Encadrant :** Dr Keba GUEYE

---

## Introduction

Ce compte rendu présente le déploiement d'une même application de scolarité dans deux architectures : un **client lourd** connecté directement à MariaDB (2-tiers), puis une **application web PHP** hébergée sur un serveur d'application qui interroge MariaDB (3-tiers). Les deux versions utilisent la même base de données. Elles sont comparées sur quatre critères : déploiement d'une évolution, cohérence des règles, volume transféré sur le réseau et sécurité.

### Les deux architectures du TP

| Architecture | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|
| **2-tiers** (client lourd) | Poste client (Windows 192.168.56.1 ou VM2 192.168.56.12) : présentation + logique métier | VM1 192.168.56.11 : serveur de données MariaDB | — |
| **3-tiers** | Navigateur (Windows) : présentation seule | VM2 192.168.56.12 : Apache + PHP, logique métier | VM1 192.168.56.11 : serveur de données MariaDB |

Flux réseau : en 2-tiers, le poste client communique en SQL avec VM1 (port 3306). En 3-tiers, le navigateur communique en HTTP avec VM2 (port 80), et seule VM2 communique en SQL avec VM1.

### Contenu du dossier

- `README.md` : ce compte rendu (réponses à Q1 à Q16) ;
- `rapport_TP2.pdf` : le même compte rendu au format PDF ;
- `images/` : les captures d'écran citées ci-dessous ;
- les fichiers utilisés pour le TP : `client_lourd.py`, le dossier `app3tiers/`, `utilisateurs_tp2.sql`, `volume.sql` et `volume_annuler.sql`. Les mots de passe des comptes de TP y ont été remplacés par `xxx` (consigne : secrets jamais versionnés).

---

## Partie 1 — L'application en 2-tiers (client lourd)

Le module `pymysql` a été installé sur Windows (`py -m pip install pymysql`), puis `client_lourd.py` a été lancé. Il a ensuite été copié sur VM2 par `scp` et lancé avec `python3`, ce qui donne deux postes clients.

![Client lourd v1.0 lancé sur Windows : connexion à 192.168.56.11:3306](images/fig01_client_lourd_windows_v10.png)

*Client lourd v1.0 lancé sur Windows : connexion à 192.168.56.11:3306*

![Relevés de ESP260001 (10,75, VALIDÉ) et ESP260011 (12,55, VALIDÉ) sur Windows](images/fig02_releves_windows.png)

*Relevés de ESP260001 (10,75, VALIDÉ) et ESP260011 (12,55, VALIDÉ) sur Windows*

![Classement sur Windows : 180 lignes, 54 ms, 4,2 Kio, 18 admis sur 30](images/fig03_classement_windows.png)

*Classement sur Windows : 180 lignes, 54 ms, 4,2 Kio, 18 admis sur 30*

![Client lourd v1.0 lancé sur VM2](images/fig04_client_lourd_vm2_v10.png)

*Client lourd v1.0 lancé sur VM2*

![Classement sur VM2 : mêmes résultats (180 lignes, 4,2 Kio, 18 / 30)](images/fig05_classement_vm2.png)

*Classement sur VM2 : mêmes résultats (180 lignes, 4,2 Kio, 18 / 30)*

### Q1. Présentation, logique métier et accès aux données : où sont-elles et sur quelle machine s'exécutent-elles ?

| Couche | Lignes de `client_lourd.py` | Contenu |
|---|---|---|
| **Présentation** | 54, 61, 63, 69, 73, 89 à 91, 93, 97, 104 à 113 | Les `print` et `input` : affichage de la liste, du relevé, du classement et du bilan, menu et saisie du matricule dans `main()`. |
| **Logique métier** | 28 à 30, 39 à 42, 71 à 72, 85 à 88, 92 | `VERSION` et `SEUIL_VALIDATION` (règle n°1), `moyenne()` pondérée par les crédits (règle n°2), calcul de la décision VALIDÉ / NON VALIDÉ, regroupement et tri du classement, décompte des admis. |
| **Accès aux données** | 22 à 26, 33 à 36, 45 à 48, 52, 58, 64 à 66, 80 à 82 | Paramètres de connexion, `connexion()`, `octets_recus()` (`SHOW SESSION STATUS`) et toutes les requêtes `SELECT`. |

Les trois couches sont mélangées dans un seul fichier, et même dans une seule fonction : `releve()` (lignes 57 à 73) enchaîne requête, affichage, calcul de la moyenne et décision. **Elles s'exécutent toutes sur le poste client** (Windows ou VM2). VM1 se contente de stocker les données et de répondre aux requêtes SQL.

### Q2. Que découvre un étudiant qui récupère `client_lourd.py`, et que peut-il faire hors du programme ?

Les lignes 22 à 26 sont en clair dans le fichier et donnent : l'adresse du serveur de base de données (192.168.56.11), le port (3306), le compte (`etudiant`), son mot de passe et le nom de la base (`scolarite`). Ces informations suffisent pour se connecter à MariaDB sans passer par le programme. Nous l'avons vérifié avec une courte commande Python, qui lit directement la table `note`.

![Connexion à la base sans client_lourd.py : la table note contient 180 lignes](images/fig06_connexion_sans_programme.png)

*Connexion à la base sans client_lourd.py : la table note contient 180 lignes*

Les droits du compte, relevés avec `SHOW GRANTS FOR 'etudiant'@'192.168.56.%'`, se limitent à **`GRANT SELECT ON scolarite.*`**. Hors du programme, l'étudiant peut donc lire toute la base, y compris les notes de tous les autres étudiants, avec n'importe quel outil. Il ne peut pas modifier les données, mais ce compte est ouvert à tout le réseau `192.168.56.%` et son mot de passe est distribué dans chaque exemplaire du programme.

### Q3. Qu'aurait-il fallu installer sur les 300 postes d'une salle informatique ?

Sur chaque poste, comme nous l'avons fait sur Windows et sur VM2, il faut :

- Python 3 ;
- le module `pymysql` (pip sous Windows, `python3-pymysql` sous Linux) ;
- le fichier `client_lourd.py`, qui contient l'adresse et le mot de passe de la base ;
- un accès réseau au port 3306 du serveur de base de données.

Soit 300 installations à faire et à vérifier. Chaque mise à jour du programme oblige à recopier le fichier sur les 300 postes : un poste oublié continue à appliquer l'ancienne version (voir la partie 3).

---

## Partie 2 — La même application en 3-tiers

Sur VM2, Apache, PHP et `php-mysql` sont installés. Le dossier `app3tiers` a été copié par `scp -r` puis placé dans `/var/www/html/scolarite`. Sur VM1, le compte de l'application a été créé avec `utilisateurs_tp2.sql`.

![Copie du dossier app3tiers vers VM2](images/fig07_copie_app3tiers.png)

*Copie du dossier app3tiers vers VM2*

![Contenu de /var/www/html/scolarite sur VM2](images/fig08_contenu_scolarite_vm2.png)

*Contenu de /var/www/html/scolarite sur VM2*

Les droits du dossier ont dû être corrigés (`chmod` 755 pour les dossiers et 644 pour les fichiers, puis `chown www-data`) : Apache renvoyait d'abord une erreur 403 parce que les fichiers copiés avec `sudo` n'étaient lisibles que par root.

![Contenu de utilisateurs_tp2.sql : compte app_scolarite@192.168.56.12, SELECT seulement](images/fig09_utilisateurs_tp2_sql.png)

*Contenu de utilisateurs_tp2.sql : compte app_scolarite@192.168.56.12, SELECT seulement*

![Compte app_scolarite créé sur VM1, hôte 192.168.56.12](images/fig10_compte_app_scolarite.png)

*Compte app_scolarite créé sur VM1, hôte 192.168.56.12*

Le droit accordé au compte, relevé avec `SHOW GRANTS`, est **`GRANT SELECT ON scolarite.*`** pour `app_scolarite@192.168.56.12`.

![Application web 3-tiers : liste des étudiants (version 1.0)](images/fig11_web_liste_v10.png)

*Application web 3-tiers : liste des étudiants (version 1.0)*

![Relevé de ESP260001 en 3-tiers : 10,75, VALIDÉ](images/fig12_web_releve_esp260001.png)

*Relevé de ESP260001 en 3-tiers : 10,75, VALIDÉ*

![Classement en 3-tiers : 18 admis sur 30, calculé par le SGBD en 30 ms](images/fig13_web_classement_v10.png)

*Classement en 3-tiers : 18 admis sur 30, calculé par le SGBD en 30 ms*

### Q4. Les deux architectures donnent-elles les mêmes résultats ? Pourquoi est-ce indispensable ?

| Mesure (version 1.0) | Client lourd Windows | Client lourd VM2 | Web 3-tiers |
|---|---|---|---|
| Moyenne de ESP260001 | 10,75 | 10,75 | 10,75 |
| Décision de ESP260001 | VALIDÉ | VALIDÉ | VALIDÉ |
| Nombre d'admis | 18 / 30 | 18 / 30 | 18 / 30 |
| Top 10 du classement | identique | identique | identique |

Les résultats sont **identiques** dans les trois cas. C'est indispensable parce que l'expérience compare des architectures : la seule chose qui doit changer est l'endroit où s'exécutent les couches. Les données (même base sur VM1) et les règles de gestion (seuil de 10, moyenne pondérée par les crédits) doivent rester identiques. Sinon, une différence de volume, de temps ou de résultat pourrait venir des données ou des règles, et non de l'architecture, et aucune conclusion ne serait valable.

### Q5. Qu'a dû installer le poste Windows pour utiliser la version 3-tiers ?

**Rien** : le navigateur déjà présent suffit. Le poste n'a besoin ni de Python, ni de pymysql, ni du programme, ni d'identifiants de base de données, et il n'a pas besoin de joindre MariaDB. En 2-tiers (Q3), chaque poste devait recevoir Python, pymysql et le programme. Pour 300 postes, le 3-tiers évite les 300 installations : le seul déploiement se fait une fois, sur VM2.

### Q6. Que renvoie la requête vers `config.php` ? Pourquoi le mot de passe n'est-il jamais envoyé ?

![curl.exe -i sur inc/config.php : statut 200 et Content-Length : 0](images/fig14_curl_config_php.png)

*curl.exe -i sur inc/config.php : statut 200 et Content-Length : 0*

La requête renvoie **`HTTP/1.1 200 OK` avec `Content-Length: 0`**, c'est-à-dire un corps vide. Apache ne renvoie pas le texte d'un fichier `.php` : il le fait **exécuter par PHP sur VM2** et ne transmet que le résultat de l'exécution. Dans `config.php`, l'exécution se limite à définir quatre constantes en mémoire (`BDD_HOTE`, `BDD_BASE`, `BDD_UTILISATEUR`, `BDD_MOT_DE_PASSE`) et n'écrit rien dans la réponse. Le code et les valeurs restent sur le serveur d'application. À l'inverse, en 2-tiers, le mot de passe (ligne 25 de `client_lourd.py`) est présent dans le fichier distribué à chaque poste.

### Q7. Combien de comptes MariaDB faut-il en 3-tiers, et en 2-tiers ?

**En 3-tiers**, un seul compte suffit, `app_scolarite@192.168.56.12`, quel que soit le nombre d'utilisateurs. Seul le serveur d'application se connecte à la base, avec le seul droit `SELECT`. Les utilisateurs n'ont aucun compte de base de données.

**En 2-tiers**, chaque poste se connecte directement à MariaDB. Soit on crée **un compte par utilisateur** (N comptes à créer, protéger et retirer), soit on utilise **un compte partagé**, comme `etudiant` ici, dont le mot de passe est distribué dans le programme à tous les postes. Dans les deux cas, le compte doit être autorisé depuis tout le réseau des clients (`192.168.56.%`) et non depuis une seule machine.

### Q8. Que trouve-t-on dans le code source de la page (Ctrl+U) ?

Le code source ne contient que du **HTML** : l'en-tête généré par `gabarit.php`, le lien vers `style.css`, le menu et le tableau dont les valeurs sont déjà remplies. On n'y trouve **aucune requête SQL** (pas de `SELECT`), **aucune règle de calcul** (pas de `SUM` ni de seuil de 10) et aucune balise PHP. Le navigateur reçoit exactement le **résultat** de l'exécution du PHP sur VM2 : une page déjà construite, avec les moyennes et les décisions calculées. La logique métier et l'accès aux données restent sur le serveur. À l'inverse, le client lourd contient tout le code (lignes 22 à 113).

---

## Partie 3 — Faire évoluer une règle métier

**Règle de la version 1.1 :** une note strictement inférieure à 8 dans un EC empêche la validation, quelle que soit la moyenne (constante `NOTE_ELIMINATOIRE = 8`).

**2-tiers.** Dans `client_lourd.py` sur VM2 uniquement, nous avons ajouté la constante `NOTE_ELIMINATOIRE`, passé `VERSION` à `"1.1"`, puis modifié la décision dans `releve()` (test de la note minimale) et le décompte des admis dans `classement()`. La version 1.0 est restée sur Windows, comme un poste non mis à jour.

**3-tiers.** Dans `/var/www/html/scolarite/inc/metier.php`, nous avons passé `VERSION_APPLI` à `'1.1'`, ajouté la constante `NOTE_ELIMINATOIRE` et créé une fonction `a_note_eliminatoire()` qui parcourt les notes d'un étudiant et renvoie `true` si l'une d'elles est inférieure à `NOTE_ELIMINATOIRE`. La décision de `releve()` n'est alors VALIDÉ que si la moyenne atteint le seuil et qu'aucune note n'est éliminatoire. Le bilan de `classement()` a été complété de la même façon : sa requête reçoit maintenant deux paramètres, `SEUIL_VALIDATION` et `NOTE_ELIMINATOIRE`.

![metier.php : VERSION_APPLI passée à '1.1'](images/fig15_metier_version.png)

*metier.php : VERSION_APPLI passée à '1.1'*

![metier.php : nouvelle fonction a_note_eliminatoire() (lignes marquées AJOUT)](images/fig16_metier_fonction_eliminatoire.png)

*metier.php : nouvelle fonction a_note_eliminatoire() (lignes marquées AJOUT)*

![metier.php : la décision de releve() tient compte de la note éliminatoire](images/fig17_metier_decision_releve.png)

*metier.php : la décision de releve() tient compte de la note éliminatoire*

![metier.php : le bilan de classement() reçoit SEUIL_VALIDATION et NOTE_ELIMINATOIRE](images/fig18_metier_bilan_classement.png)

*metier.php : le bilan de classement() reçoit SEUIL_VALIDATION et NOTE_ELIMINATOIRE*

![Contrôle de la syntaxe de metier.php après modification (php -l)](images/fig19_php_syntaxe.png)

*Contrôle de la syntaxe de metier.php après modification (php -l)*

![Web 3-tiers v1.1 : ESP260011 passe à NON VALIDÉ (7,50 en MAT) malgré une moyenne de 12,55](images/fig20_web_v11_releve_esp260011.png)

*Web 3-tiers v1.1 : ESP260011 passe à NON VALIDÉ (7,50 en MAT) malgré une moyenne de 12,55*

![Web 3-tiers v1.1 : classement, 16 admis sur 30](images/fig21_web_v11_classement.png)

*Web 3-tiers v1.1 : classement, 16 admis sur 30*

### Q9. Décision pour ESP260011 et nombre d'admis sur chaque poste en 2-tiers : que constate-t-on ?

| Mesure | Windows (client v1.0) | VM2 (client v1.1) |
|---|---|---|
| Décision de ESP260011 (moyenne 12,55) | VALIDÉ | NON VALIDÉ (7,50 en MAT) |
| Nombre d'admis | 18 / 30 | 16 / 30 |

Pour la **même base de données**, les deux postes donnent des décisions et un nombre d'admis **différents** dès que l'un des deux n'est pas mis à jour : le poste Windows (v1.0) déclare ESP260011 VALIDÉ et compte 18 admis, alors que VM2 (v1.1) le déclare NON VALIDÉ à cause de sa note de 7,50 en MAT et ne compte que 16 admis. L'application web, passée en v1.1, donne les mêmes résultats que VM2.

**Conséquences pour un service de scolarité :** un même étudiant serait validé ou refusé selon le poste utilisé ; les procès-verbaux et les statistiques d'admission ne seraient pas fiables ; des contestations et des litiges seraient possibles, et il serait difficile de savoir quelle décision fait foi.

### Q10. Combien de fichiers et de machines modifiés en 3-tiers ? Combien d'utilisateurs ont bénéficié de la règle, et quand ?

**Un seul fichier** (`inc/metier.php`) sur **une seule machine** (VM2). Les pages `releve.php`, `classement.php`, `index.php`, `gabarit.php` et `config.php` n'ont pas changé. **Tous les utilisateurs** ont bénéficié de la nouvelle règle **immédiatement**, dès le rechargement de la page, sans rien installer chez eux : le poste Windows, qui n'a pas été touché, affiche la version 1.1 dans l'en-tête de la page.

### Q11. Où a-t-il fallu intervenir pour la règle éliminatoire dans chaque cas ?

**En 3-tiers**, la moyenne est calculée par le SGBD (`SQL_MOYENNE = SUM(valeur × crédits) / SUM(crédits)`). La règle s'est donc ajoutée dans `metier.php` : une nouvelle fonction `a_note_eliminatoire()` et la constante `NOTE_ELIMINATOIRE`, puis deux interventions, dans `releve()` (la décision appelle la nouvelle fonction) et dans `classement()` (la requête du bilan tient compte de la note minimale de chaque étudiant, avec `NOTE_ELIMINATOIRE` comme second paramètre).

**En 2-tiers**, la moyenne est calculée en Python par `moyenne()`. La règle s'est ajoutée à deux endroits de `client_lourd.py` : dans `releve()` (note minimale) et dans `classement()` (note minimale de chaque étudiant dans `par_etudiant` pour le décompte). La différence essentielle est que ces modifications doivent être refaites **sur chaque poste**, alors qu'en 3-tiers elles ne sont faites qu'une fois.

---

## Partie 4 — Volume de données et réseau

Le script `volume.sql` a été exécuté sur VM1 : la base contient alors 20 030 étudiants et 120 180 notes.

![Après volume.sql : 20 030 étudiants et 120 180 notes](images/fig22_volume_sql.png)

*Après volume.sql : 20 030 étudiants et 120 180 notes*

![Classement en 2-tiers avec le volume : 120 180 lignes, 469 ms, 2 407,3 Kio](images/fig23_classement_client_volume.png)

*Classement en 2-tiers avec le volume : 120 180 lignes, 469 ms, 2 407,3 Kio*

![Classement en 3-tiers avec le volume : calculé par le SGBD en 610 ms](images/fig24_classement_web_volume.png)

*Classement en 3-tiers avec le volume : calculé par le SGBD en 610 ms*

Le nombre d'admis diffère entre les deux captures (9 407 / 20 030 en 2-tiers, 16 / 20 030 en 3-tiers) pour une raison qui n'a rien à voir avec l'architecture : le client Windows est resté en version 1.0, alors que l'application web applique la règle de la version 1.1 (note éliminatoire). C'est une nouvelle illustration de la question Q9. Seuls le volume, le temps et la machine de calcul sont comparés ci-dessous.

| Mesure | 2-tiers | 3-tiers |
|---|---|---|
| Lignes ou octets reçus par le poste client | 120 180 lignes, 2 407,3 Kio | Environ 1,5 Kio (une page HTML de 10 lignes) |
| Temps de calcul du classement | 469 ms (requête et réception des lignes) | 610 ms (calcul par le SGBD, affiché par la page) |
| Machine qui a effectué le calcul | Le poste client (Windows) | VM1 : le SGBD (VM2 ne fait que la mise en page) |

Pour référence, avant l'ajout du volume (30 étudiants, 180 notes) : le client lourd recevait 180 lignes (4,2 Kio) en 54 ms sur Windows et 5 ms sur VM2, et la page 3-tiers affichait 30 ms puis 11 ms. Les premières exécutions sont plus lentes (connexion et cache).

L'ajout du volume a été annulé ensuite avec `volume_annuler.sql` : la table `note` est revenue à 180 lignes.

![Annulation du volume : retour à 180 notes](images/fig25_volume_annuler.png)

*Annulation du volume : retour à 180 notes*

### Q12. D'où vient l'écart de volume ? Que deviendrait-il pour 300 postes à 10 Mbit/s, ou sur une connexion mobile ?

L'écart vient de **l'endroit où se fait le calcul**. Le client lourd rapatrie **toutes les notes brutes** (120 180 lignes, soit 2 407,3 Kio) pour calculer lui-même les moyennes, puis n'en affiche que 10. En 3-tiers, c'est le SGBD qui calcule : seules 10 lignes de classement et une ligne de bilan quittent VM1, et le navigateur ne reçoit qu'une page HTML déjà prête.

2 407,3 Kio représentent environ 19,7 Mbit par poste. Pour **300 postes** qui demandent le classement sur une liaison partagée de 10 Mbit/s : 300 × 19,7 ≈ 5 900 Mbit, soit **près de 10 minutes** de saturation du lien, alors qu'un poste seul met environ 2 secondes. En 3-tiers, la page envoyée à chaque poste pèse environ 1,5 Kio (une dizaine de lignes de tableau), soit plus de 1 000 fois moins, et cette taille ne dépend pas du volume de la base. Sur une **connexion mobile**, chaque classement consommerait près de 2,4 Mo de données facturées, avec des temps d'attente importants. Cela s'aggrave avec le nombre d'étudiants et de notes : le trafic du 2-tiers croît avec le volume de la base, celui du 3-tiers reste proche de la taille de la page affichée.

### Q13. Que mesure `Bytes_sent` ? Pourquoi est-ce une mesure honnête du trafic ?

Dans `client_lourd.py` (lignes 45 à 48 et 78 à 84), le client exécute `SHOW SESSION STATUS LIKE 'Bytes_sent'` avant et après la requête du classement, puis fait la différence. **`Bytes_sent` est un compteur tenu par le serveur MariaDB : le nombre d'octets qu'il a envoyés au client sur cette connexion.** C'est une mesure honnête parce qu'elle compte les octets réellement émis par le serveur, sans estimation côté client ; le mot `SESSION` limite la mesure à la connexion du client, si bien que le trafic des autres utilisateurs n'entre pas dans le résultat. La requête `SHOW STATUS` elle-même ajoute quelques octets, négligeables devant 2 407 Kio.

---

## Partie 5 — Sécurité : fermer l'accès direct à la base

Sur VM1, le pare-feu a été configuré pour n'autoriser le port 3306 que depuis VM2, en autorisant d'abord SSH pour ne pas perdre l'accès à la machine :

![ufw allow OpenSSH](images/fig26_ufw_openssh.png)

*ufw allow OpenSSH*

![ufw allow 80/tcp](images/fig27_ufw_80.png)

*ufw allow 80/tcp*

![ufw allow from 192.168.56.12 to any port 3306](images/fig28_ufw_allow_vm2.png)

*ufw allow from 192.168.56.12 to any port 3306*

![ufw deny 3306/tcp](images/fig29_ufw_deny_3306.png)

*ufw deny 3306/tcp*

![ufw enable : pare-feu actif](images/fig30_ufw_enable.png)

*ufw enable : pare-feu actif*

![État du pare-feu : le port 3306 n’est autorisé que depuis 192.168.56.12](images/fig31_ufw_status.png)

*État du pare-feu : le port 3306 n’est autorisé que depuis 192.168.56.12*

![Client lourd depuis Windows après le pare-feu : connexion impossible (timed out)](images/fig32_client_timeout.png)

*Client lourd depuis Windows après le pare-feu : connexion impossible (timed out)*

### Q14. Que se passe-t-il pour chaque architecture ? Pourquoi ne peut-on pas appliquer cette protection en 2-tiers ?

**En 2-tiers**, le client lourd lancé sur Windows ne peut plus joindre MariaDB : après le délai de 5 secondes, il affiche « Connexion impossible au serveur de base de données : (2003, … timed out) ». La règle est de type `deny` : les paquets sont ignorés, sans réponse.

**En 3-tiers**, l'application web continue de fonctionner : Windows communique avec VM2 sur le port 80, et c'est VM2, seule adresse autorisée, qui communique avec VM1 sur le port 3306.

Cette protection est impossible en 2-tiers parce que **chaque poste client doit se connecter directement au port 3306** pour que l'application fonctionne. Il faudrait ouvrir le port à toutes les adresses des postes (300 adresses ou tout un réseau), ce qui revient à exposer la base à tous ceux qui disposent d'un poste. En 3-tiers, une seule machine a besoin de ce port : VM2. À noter que le pare-feu filtre par adresse de machine et non par programme : un client lourd lancé depuis VM2 resterait autorisé.

### Q15. Deux autres risques de sécurité propres au client lourd, et comment le 3-tiers les supprime ou les réduit.

- **Les identifiants de la base sont distribués avec le programme.** L'adresse, le compte et le mot de passe sont en clair dans `client_lourd.py` (lignes 22 à 26) : quiconque a le fichier peut se connecter à la base avec un autre outil (Q2). **Avec le 3-tiers** : ils restent dans `config.php` sur VM2 et ne sont jamais envoyés (Q6) ; les utilisateurs n'ont aucun compte de base.
- **La logique métier s'exécute chez l'utilisateur.** Il peut modifier le programme, ou écrire le sien, pour changer le seuil, les moyennes ou les décisions, et la base ne peut pas faire la différence ; les postes peuvent aussi appliquer des versions différentes (Q9). **Avec le 3-tiers** : la règle s'exécute sur VM2, hors de portée des utilisateurs, et le compte `app_scolarite` n'a que le droit `SELECT`.

On peut ajouter la surface d'attaque réseau : en 2-tiers, le port de la base est ouvert à tout le réseau des clients et les requêtes SQL circulent entre les postes et le serveur ; en 3-tiers, il n'est ouvert qu'à VM2.

---

## Synthèse

| Critère | 2-tiers | 3-tiers |
|---|---|---|
| **Logiciel à installer sur chaque poste** | Python 3, pymysql et `client_lourd.py`, avec accès réseau au port 3306 | Un navigateur seulement |
| **Déploiement d'une nouvelle règle** | Modifier le programme sur chaque poste ; ici, un seul poste sur deux mis à jour | Un fichier (`metier.php`) sur une machine (VM2), effet immédiat pour tous |
| **Cohérence entre utilisateurs** | Risque de versions différentes : ESP260011 VALIDÉ en v1.0, NON VALIDÉ en v1.1 ; 18 admis contre 16 | Une seule copie de la règle : mêmes résultats pour tous |
| **Trafic réseau côté client** | 120 180 lignes, 2 407,3 Kio pour un classement | Environ 1,5 Kio : une page HTML de 10 lignes |
| **Exposition du SGBD et des mots de passe** | Port 3306 ouvert à tous les postes ; identifiants dans le programme | Port 3306 réservé à VM2 ; mot de passe sur VM2 seulement, jamais envoyé |
| **Charge des serveurs** | VM1 envoie toutes les notes ; le calcul est fait sur les postes | VM1 calcule (agrégation SQL), VM2 génère les pages : la charge est concentrée sur les serveurs |
| **Simplicité de développement** | Un seul programme, pas de serveur web : plus simple à écrire | Plusieurs couches et fichiers, serveur d'application à installer et à maintenir |

### Q16. Conditions d'acceptabilité du 2-tiers, et justification du 3-tiers pour toute l'université.

**Le 2-tiers est acceptable** dans des cas comme le logiciel de caisse d'une boutique, le poste de saisie d'un laboratoire ou un outil d'administration de base de données, lorsque les conditions suivantes sont réunies :

- un **petit nombre de postes** (un ou quelques-uns), donc un déploiement et des mises à jour simples à contrôler ;
- un **réseau local de confiance**, où le port de la base n'est pas exposé à des personnes non autorisées ;
- des **utilisateurs de confiance**, souvent les administrateurs eux-mêmes, qui ont de toute façon le droit d'accéder à la base ;
- des **données peu sensibles** ou des droits très limités, et un volume de données modeste qui ne sature pas le réseau ;
- des **règles de gestion stables** ou des mises à jour rares.

**Pour un service de scolarité utilisé par toute l'université, le 3-tiers s'impose :**

- **Déploiement :** des milliers d'utilisateurs n'ont besoin que d'un navigateur, et une évolution des règles se fait en un seul endroit (partie 3).
- **Cohérence :** tous les utilisateurs appliquent la même règle, ce qui est indispensable pour des décisions de validation et des procès-verbaux (Q9).
- **Sécurité :** les notes sont des données sensibles ; avec le 3-tiers, la base n'est joignable que depuis le serveur d'application, aucun mot de passe de base n'est distribué, et chaque utilisateur peut être authentifié par l'application (partie 5).
- **Volume et réseau :** seules les données utiles circulent vers les utilisateurs, y compris sur des liaisons lentes ou mobiles (Q12).

Le prix à payer est une architecture plus complexe, avec un serveur d'application à installer, à sécuriser et à maintenir, mais ce coût est largement compensé à cette échelle.
