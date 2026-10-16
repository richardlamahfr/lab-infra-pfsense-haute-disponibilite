# Lab d'infrastructure virtualisée : pfSense HA, Active Directory, Zabbix

Infrastructure de production simulée de 11 machines virtuelles sous VMware Workstation, conçue pour reproduire le système d'information d'une petite entreprise : pare-feu redondants, annuaire Active Directory, serveur web et supervision centralisée.

![Architecture réseau](docs/images/architecture-reseau.png)

## Compétences mises en œuvre

- **Réseau et sécurité** : deux pfSense 2.7.2 en haute disponibilité CARP (VIP LAN 192.168.99.100 et VIP WAN 192.168.112.99, synchronisation sur un réseau dédié, bascule mesurée à environ 3 secondes), NAT, filtrage de la sortie Internet par plage d'adresses
- **Windows Server** : contrôleur de domaine Active Directory et DNS (Windows Server 2019), quatre postes Windows 10 joints au domaine
- **Linux** : Debian 13 et Ubuntu Server 24.04, pile web Nginx, MariaDB et WordPress
- **Supervision** : Zabbix Server avec base MariaDB, agents Zabbix sur les serveurs Linux
- **Documentation** : dossier d'architecture, plan de validation, procédures d'incident

## Architecture

| Zone | Réseau | Segment VMware | Rôle |
|---|---|---|---|
| WAN | 192.168.112.0/24 | NAT | Sortie Internet par la passerelle 192.168.112.2, VIP CARP WAN 192.168.112.99 |
| LAN | 192.168.99.0/24 | VMnet14 | Serveurs et postes, passerelle virtuelle 192.168.99.100 |
| Synchronisation | 10.1.10.0/24 | VMnet15 | Synchronisation de la paire pfSense |

| Machine | Système | Adresse | Rôle |
|---|---|---|---|
| pfSense 1 / pfSense 2 | pfSense 2.7.2 | 192.168.99.1 / .2 | Pare-feu actif / secours |
| Serveur AD | Windows Server 2019 | 192.168.99.15 | Active Directory, DNS |
| 4 postes | Windows 10 | .6, .7, .10, .20 | Direction, Développement, Comptabilité, Marketing |
| Serveur Zabbix | Debian 13 | 192.168.99.30 | Zabbix Server, MariaDB |
| Serveur web | Ubuntu Server 24.04 | 192.168.99.16 | WordPress, Nginx, MariaDB |
| Kali | Kali Linux | 192.168.99.25 | Tests de sécurité |
| Client Linux | Ubuntu | 192.168.99.8 | Poste Linux |

![Haute disponibilité](docs/images/haute-disponibilite.png)

## Tests de validation (16 octobre 2026)

| ID | Test | Résultat attendu | Résultat obtenu |
|---|---|---|---|
| Référence | Les deux pfSense arrêtés | Aucune sortie Internet | Conforme : « Destination Host Unreachable » sur Ubuntu, pas d'Internet sur Windows Server |
| HA 01 | pfSense 1 arrêté, pfSense 2 seul | La passerelle 192.168.99.100 reste disponible | **Réussi** : ping 8.8.8.8 sans perte (28 à 39 ms), navigation web depuis Windows Server |
| NET 02 | Sortie Internet depuis le LAN | Accès fonctionnel | **Réussi** |
| NET 03 | Sortie Internet depuis 192.168.99.110 à .120 | Bloquée, LAN joignable | **Réussi** |
| HA 01b | Arrêt brutal de pfSense 1 pendant un ping continu | Coupure mesurée | **Réussi : environ 3 s** (3 paquets perdus sous Linux, 1 délai d'attente sous Windows) |
| HA 03 | Arrêt des deux pfSense | Coupure totale, reprise dès qu'un nœud revient | **Conforme** |
| HA 02 | Retour de pfSense 1 | pfSense 1 redevient MASTER | **Réussi** : pfSense 2 repasse BACKUP automatiquement |
| HA 04 | Synchronisation des états (pfsync) | Les deux nœuds partagent leurs états | **Réussi** après correction de l'adresse du pair pfsync |
| HA 05 | Session SSH ouverte pendant une bascule | La session survit | À réaliser |

**Sans pare-feu, aucune sortie :**

![Ubuntu sans pfSense](docs/images/tests/03-ubuntu-ping-sans-pfsense.png)

**pfSense 1 arrêté : pfSense 2 reprend seul le trafic :**

![Console pfSense 2](docs/images/tests/01-console-pfsense2-interfaces.png)

![Ubuntu via pfSense 2](docs/images/tests/04-ubuntu-ping-via-pfsense2.png)

**Bascule à chaud : pfSense 1 arrêté en plein trafic.** Les requêtes 55, 56 et 57 n'ont pas de réponse (sous Linux, ping n'affiche pas les pertes, elles se lisent dans les numéros manquants), puis le trafic repart par pfSense 2 :

![Bascule à chaud](docs/images/tests/07-bascule-arret-pfsense1-ubuntu.png)

**Statut CARP : pfSense 2 devient MASTER quand pfSense 1 tombe, puis repasse BACKUP à son retour :**

![pfSense 2 MASTER](docs/images/tests/12-carp-pfsense2-devient-master.png)

![pfSense 2 BACKUP](docs/images/tests/13-carp-pfsense2-repasse-backup.png)

Détail de la campagne et captures complémentaires : sections 11 à 11.4 du [dossier d'architecture](docs/dossier-architecture.md#11-plan-de-validation).

## Problèmes rencontrés et solutions

### 1. Les deux pare-feu se déclaraient maîtres (split-brain CARP)

**Symptôme** : une fois la haute disponibilité configurée, pfSense 1 et pfSense 2 affichaient tous les deux le statut MASTER pour la passerelle virtuelle 192.168.99.100, et le trafic du LAN ne sortait plus.

**Cause** : avec CARP, le nœud de secours ne reste BACKUP que s'il reçoit les annonces du maître. Ici, chaque nœud ne recevait pas les annonces de l'autre et se proclamait maître. Deux équipements répondaient alors pour la même adresse 192.168.99.100, et le trafic ne pouvait plus sortir correctement.

**Résolution** : reprise de la configuration CARP sur les deux nœuds, en s'appuyant sur la documentation officielle de pfSense et sur des forums décrivant le même cas, jusqu'à obtenir un seul maître et un nœud de secours en BACKUP.

**Résultat** : un seul maître à la fois ; quand pfSense 1 est arrêté, pfSense 2 assure seul la sortie Internet (test HA 01).

### 2. Pas de sortie Internet depuis le LAN

**Symptôme** : les machines du LAN n'accédaient pas à Internet.

**Cause** : la règle de pare-feu autorisant la sortie sur l'interface LAN de pfSense était mal définie.

**Résolution** : correction de la règle de sortie, en conservant le blocage de la plage 192.168.99.110 à .120, qui doit rester joignable en interne sans sortir sur Internet.

**Résultat** : sortie autorisée pour le LAN et bloquée pour la plage réservée (tests NET 02 et NET 03).

### 3. Une machine hôte de 16 Go de RAM pour 11 VM

**Problème** : avec 16 Go de RAM, le PC hôte ne peut pas faire tourner les 11 machines virtuelles en même temps.

**Résolution** : pour chaque scénario de test, seules les VM nécessaires sont démarrées. Le test HA 01, par exemple, a été réalisé avec pfSense 2, Windows Server 2019, Ubuntu Client et un poste Windows.

**Résultat** : tous les tests ont pu être menés malgré les ressources limitées.

### 4. Une passerelle WAN mal documentée

**Symptôme** : la première version du dossier présentait 192.168.112.99 comme la passerelle Internet amont des deux pare-feu.

**Cause** : en relisant le statut CARP, 192.168.112.99 apparaît comme la VIP CARP WAN (VHID 1), une adresse portée par la paire pfSense elle-même, donc pas sa passerelle.

**Résolution** : relevé de la vraie passerelle dans System > Routing > Gateways : 192.168.112.2, la passerelle du NAT VMware. Correction du plan d'adressage, des procédures et des schémas.

**Ce que j'en retiens** : une documentation se vérifie sur l'équipement, pas de mémoire.

### 5. Synchronisation des états pfsync inopérante

**Symptôme** : dans Status > CARP, chaque nœud n'affichait que son propre identifiant d'états (6a813fb7 sur pfSense 1, 3280341d sur pfSense 2). pfSense indique que les deux identifiants doivent apparaître sur chaque nœud quand pfsync fonctionne.

**Impact** : CARP basculait bien la VIP, mais le nœud de secours ne connaissait pas les connexions en cours. Un ping repart, une session TCP (SSH, RDP, téléchargement) aurait été coupée à la bascule.

**Diagnostic** :
1. Ajout d'une règle autorisant tout le trafic sur l'interface SYNC : nécessaire, mais le défaut persiste.
2. Vérification de System > High Avail Sync : le champ **pfsync Synchronize Peer IP** contenait **192.168.99.1**, l'adresse LAN de pfSense 1. Les états partaient vers une adresse hors du réseau de synchronisation.

**Résolution** : adresse du pair placée sur le réseau SYNC (10.1.10.3 sur pfSense 1, 10.1.10.2 sur pfSense 2) ; synchronisation de configuration XMLRPC configurée uniquement sur le maître.

**Résultat** : chaque nœud affiche les deux identifiants, les tables d'états sont partagées (test HA 04).

![pfsync fonctionnel sur pfSense 1](docs/images/tests/20-pfsync-ok-pfsense1.png)

**Ce que j'en retiens** : quand CARP bascule correctement, cela ne prouve pas que pfsync fonctionne ; les deux se vérifient séparément.

## Évolutions prévues

- Test de survie d'une session SSH pendant une bascule (HA 05)
- WAN des deux nœuds en adresse fixe plutôt qu'en DHCP
- Segmentation en VLAN (serveurs, utilisateurs, administration, DMZ pour le serveur web)
- Second contrôleur de domaine et DNS
- Sauvegarde des VM et des configurations, avec test de restauration (RPO et RTO mesurés)
- Supervision du serveur Windows par Zabbix

## Documentation

- [Dossier d'architecture technique complet](docs/dossier-architecture.md)

## Auteur

Richard Lamah, administrateur systèmes et réseaux / ingénieur virtualisation · [LinkedIn](https://www.linkedin.com/in/richard-lamah)
