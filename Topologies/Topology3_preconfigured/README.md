# Topologie 3 (Préconfigurée) - Routage Dynamique RIPv2 (OVS + FRR)

## Objectif du Lab
Ce scénario déploie une infrastructure réseau avancée de niveau 3 dite "Pur SDN". L'objectif est de valider le fonctionnement d'un protocole de routage dynamique multi-sauts (RIPv2) au sein d'un environnement strictement conteneurisé. 

L'architecture est composée de trois domaines de diffusion distincts portés par Open vSwitch (OVS) :
* **VLAN 10 :** Réseau d'accès du Client (10.1.10.0/24).
* **VLAN 20 :** Réseau d'accès du Serveur (10.1.20.0/24).
* **VLAN 30 :** Réseau de Transit reliant les deux routeurs (10.1.30.0/24).

Pour garantir un comportement réaliste et éviter les interférences du noyau hôte ou de Docker (collisions d'adresses MAC, règles Iptables parasites), les nœuds sont totalement isolés des réseaux Docker natifs et leurs adresses MAC sont fixées de manière déterministe. Une règle de Port Mirroring (SPAN) est configurée sur le réseau de Transit (VLAN 30) pour observer les échanges de tables de routage (RIP) et le trafic utilisateur (ICMP).

## Rôles des Conteneurs
L'infrastructure déploie 5 nœuds distincts :
* **Client :** Machine d'extrémité (VLAN 10) utilisée pour initier les tests de connectivité (Ping).
* **Serveur :** Machine cible (VLAN 20) répondant aux requêtes.
* **Router 1 (R1) :** Équipement FRRouting personnalisé (`frr-rip-img`) avec le démon RIP activé. Il connecte le VLAN 10 au VLAN 30 de transit.
* **Router 2 (R2) :** Équipement FRRouting identique connectant le VLAN 20 au VLAN 30 de transit.
* **Sonde :** Nœud d'observation écoutant le trafic dupliqué depuis le VLAN 30.

## 1. Exécution du Déploiement Automatisé
Le script `setup_solution3Lab.sh` gère l'intégralité du cycle de vie : nettoyage, isolation Pur SDN (déconnexion du bridge Docker, forçage des MACs, désactivation du Proxy ARP), câblage OVS, purge des règles pare-feu, et configuration de RIPv2 via `vtysh`. 

À la fin de son exécution, le script marque une pause de 15 secondes (convergence RIP accélérée), lance la capture réseau, et effectue un `ping` automatisé de bout en bout.

Pour lancer l'infrastructure et le test, placez-vous dans le dossier et exécutez :
```bash
chmod +x setup_solution3Lab.sh
./setup_solution3Lab.sh
```

## 2. Visualisation du Trafic Capturé
Le test génère un fichier `.pcap` contenant la preuve que les routeurs ont échangé leurs tables RIP et que le ping a traversé le réseau de transit. Vous pouvez analyser ce trafic de deux manières :

### Méthode A : En ligne de commande depuis la Sonde
Vous pouvez lire le fichier de capture directement depuis l'intérieur du conteneur sonde :
```bash
docker exec -it sonde tcpdump -r /data/capture_topology3_rip.pcap -n
```

### Méthode B : Via l'interface graphique Wireshark sur la VM Debian
Le fichier de capture est automatiquement sauvegardé sur votre machine hôte grâce aux volumes Docker.
1. Ouvrez **Wireshark** sur votre machine Debian.
2. Allez dans **Fichier > Ouvrir** et naviguez vers : `/home/debian/PRES/captures_trafic/trafic_topo3/capture_topology3_rip.pcap`
3. Appliquez le filtre `rip` pour observer les paquets *Response Multicast* (224.0.0.9) échangés par les routeurs.
4. Appliquez le filtre `icmp` pour vérifier que les *Echo Request* et *Echo Reply* ont bien circulé entre 10.1.10.2 et 10.1.20.2.

## 3. Expérimentation et Vérification en Temps Réel
Pour valider le bon fonctionnement de la solution, il est indispensable d'inspecter les tables de routage avant d'effectuer un test de trafic en direct.

**Étape 1 : Vérification de la Convergence (Tables de routage)**
Sur votre terminal Debian, interrogez les tables de routage générées par Zebra/RIP dans les conteneurs :
```bash
docker exec router1 ip route
docker exec router2 ip route
```
**Résultat attendu :** Sur `router1`, vous devez voir une ligne indiquant que le réseau `10.1.20.0/24` a été appris via le protocole `rip` en passant par la passerelle `10.1.30.2`.

**Étape 2 : Lancement de l'écoute réseau (Terminal 1)**
Sur votre machine Debian, lancez Wireshark en super-utilisateur pour écouter le câble virtuel de transit attaché à R1 :
```bash
sudo wireshark -i veth-r1-30-host -k
```
*(Appliquez le filtre `icmp or rip` pour observer à la fois les mises à jour de routage et les pings).*

**Étape 3 : Génération du trafic manuel (Terminal 2)**
Ouvrez un second terminal, connectez-vous au conteneur client et lancez un ping continu :
```bash
docker exec -it client bash
ping 10.1.20.2
```
**Résultat attendu :** Dans le terminal du client, les réponses ICMP s'affichent sans perte. Dans Wireshark, vous verrez les paquets ICMP encapsulés avec un tag VLAN 30 (`802.1Q Virtual LAN, PRI: 0, DEI: 0, ID: 30`) traverser le lien de transit en temps réel, entrecoupés toutes les 5 secondes par les annonces de routage RIPv2.
