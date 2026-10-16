# Lab d'infrastructure virtualisée : pfSense HA, Active Directory, Zabbix

Infrastructure de production simulée de 11 machines virtuelles sous VMware Workstation, conçue pour reproduire le système d'information d'une petite entreprise : pare-feu redondants, annuaire Active Directory, serveur web et supervision centralisée.

![Architecture réseau](docs/images/architecture-reseau.png)

## Compétences mises en œuvre

- **Réseau et sécurité** : deux pfSense 2.7.2 en haute disponibilité (passerelle virtuelle 192.168.99.100, synchronisation sur un réseau dédié, bascule mesurée à environ 3 secondes), NAT, filtrage de la sortie Internet par plage d'adresses
- **Windows Server** : contrôleur de domaine Active Directory et DNS (Windows Server 2019), quatre postes Windows 10 joints au domaine
- **Linux** : Debian 13 et Ubuntu Server 24.04, pile web Nginx, MariaDB et WordPress
- **Supervision** : Zabbix Server avec base MariaDB, agents Zabbix sur les serveurs Linux
- **Documentation** : dossier d'architecture, plan de validation, procédures d'incident

## Architecture

| Zone | Réseau | Segment VMware | Rôle |
|---|---|---|---|
| WAN | 192.168.112.0/24 | NAT | Sortie Internet des deux pfSense |
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
| HA 02 | Retour du nœud initial | Configuration synchronisée | À réaliser |

**Sans pare-feu, aucune sortie :**

![Ubuntu sans pfSense](docs/images/tests/03-ubuntu-ping-sans-pfsense.png)

**pfSense 1 arrêté : pfSense 2 reprend seul le trafic :**

![Console pfSense 2](docs/images/tests/01-console-pfsense2-interfaces.png)

![Ubuntu via pfSense 2](docs/images/tests/04-ubuntu-ping-via-pfsense2.png)

**Bascule à chaud : pfSense 1 arrêté en plein trafic.** Les requêtes 55, 56 et 57 n'ont pas de réponse (sous Linux, ping n'affiche pas les pertes, elles se lisent dans les numéros manquants), puis le trafic repart par pfSense 2 :

![Bascule à chaud](docs/images/tests/07-bascule-arret-pfsense1-ubuntu.png)

Détail de la campagne et captures complémentaires : section 11 du [dossier d'architecture](docs/dossier-architecture.md#11-plan-de-validation).

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

## Évolutions prévues

- Test du retour du nœud initial (HA 02) et réduction du temps de bascule
- Segmentation en VLAN (serveurs, utilisateurs, administration, DMZ pour le serveur web)
- Second contrôleur de domaine et DNS
- Sauvegarde des VM et des configurations, avec test de restauration (RPO et RTO mesurés)
- Supervision du serveur Windows par Zabbix

## Documentation

- [Dossier d'architecture technique complet](docs/dossier-architecture.md)

## Auteur

Richard Lamah, administrateur systèmes et réseaux / ingénieur virtualisation · [LinkedIn](https://www.linkedin.com/in/richard-lamah)
