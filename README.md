# Docker Netlab - Sorbonne Université

## Présentation du Projet
Ce projet, réalisé dans le cadre du Master 1 (Parcours RES) à Sorbonne Université, propose une solution légère et isolée pour l'expérimentation réseau. L'objectif est de remplacer les infrastructures matérielles lourdes (routeurs Cisco, switchs physiques) par un environnement conteneurisé performant basé sur **Docker**, **Open vSwitch (OVS)** et **FRRouting (FRR)**.

Le projet permet de simuler des topologies allant de la simple commutation de niveau 2 (L2) au routage inter-VLAN de niveau 3 (L3), tout en intégrant des fonctionnalités avancées d'ingénierie de trafic comme le **Port Mirroring (SPAN)** et l'émulation de contraintes réseau (latence, perte de paquets).

## Structure du Dépôt
Le répertoire est organisé pour séparer les infrastructures réseaux (Topologies) des scénarios applicatifs (Labs) :

```text
.
├── Topologies/
│   ├── Topology1_InternetRequired/    # Topologie L2 (Switching) - Dépendances apt au runtime [cite: 36, 44]
│   ├── Topology2_InternetRequired/    # Topologie L3 (Routing) - Dépendances apt au runtime [cite: 70, 79]
│   └── Topology2_preconfigured/       # Topologie L3 préconfigurée pour usage hors-ligne [cite: 54]
│
├── Labs/
│   ├── ftp_scenario/                  # Test de transfert FTP sécurisé sur topologie L3 [cite: 3, 16]
│   └── ssh_scenario/                  # Test de connexion SSH/Telnet sur topologie L3 [cite: 22, 34]
│
├── captures_trafic/                   # Répertoire de stockage des captures .pcap (Topo 1, 2 et 3)
└── setup_solution.sh                  # Scripts d'orchestration et de câblage OVS/Veth [cite: 3, 22, 45, 80]

## Types de Topologies
Le projet propose deux approches de déploiement :

1. **Topologies avec accès Internet :** Les dépendances logicielles (`iproute2`, `tcpdump`, etc.) sont téléchargées et installées dynamiquement lors de l'exécution du script via `apt update`.
2. **Topologies et Labs préconfigurés :** Ces versions utilisent des images Docker construites localement en amont via des `Dockerfile`. Elles sont indispensables pour les environnements isolés (comme les postes en salles de TP sans accès internet externe) car elles embarquent nativement tous les outils réseau nécessaires.


## Utilisation des Environnements Préconfigurés (Hors-Ligne / Salles de TP)

Cette section détaille la marche à suivre pour déployer les topologies et laboratoires préconfigurés (`Topology2_preconfigured`, `ftp_scenario`, `ssh_scenario`) dans un environnement sans accès à Internet.

Il existe deux approches pour obtenir les images Docker requises : la construction locale via les `Dockerfile` fournis (nécessite Internet une seule fois), ou le chargement direct depuis une archive `.tar`.

### Option A : Construction locale des images (Nécessite Internet)

Si vous disposez d'une connexion Internet temporaire, vous pouvez construire les images manuellement. [cite_start]Les noms d'images ci-dessous correspondent à ceux renseignés dans les fichiers `docker-compose.yaml` du projet[cite: 16, 19, 20, 53].

**1. Pour la Topologie 2 préconfigurée :**
Depuis `Topologies/Topology2_preconfigured/Dockerfiles_List/` :
```bash
# Image Client
docker build -t client-img:latest -f client/Dockerfile .
# Image Serveur
docker build -t server-img:latest -f server/Dockerfile .
# Image Sonde
docker build -t sonde-img:latest -f sonde/Dockerfile .
```

**2. Pour les scénarios applicatifs (Labs) :**
Depuis les répertoires respectifs (`Labs/ftp_scenario/` ou `Labs/ssh_scenario/`) :
```bash
# Exemple pour le Lab FTP
docker build -t ftp_scenario-server:latest -f Dockerfile.tme .
docker build -t ftp_scenario-client:latest -f Dockerfile.tme .
```

### Option B : Flux de déploiement Hors-Ligne (Save & Load)

Pour les machines totalement isolées, utilisez la méthode d'archivage.

**1. Exportation (Sur une machine connectée) :**
Après avoir construit les images, regroupez-les dans une archive unique :
```bash
docker save -o images_netlab.tar \
  client-img:latest server-img:latest sonde-img:latest \
  frrouting/frr:latest \
  ftp_scenario-server:latest ftp_scenario-client:latest \
  ssh_scenario-server:latest ssh_scenario-client:latest
```
> IMPORTANT (contexte d'exécution sur les machines de la PPTI):
> La VM installée sur les machines PPTI possède dèjà les archives `.tar` (qui ne sont pas présents dans ce répo GitHub),
> il n'est donc pas nécessaire d'exécuter l'étape précédente (sauf à des fins de test additionnel).

**2. Chargement (Sur la machine cible isolée) :**
Transférez le fichier `images_netlab.tar` et chargez-le :
```bash
docker load -i images_netlab.tar
```
Pour vérifier les images chargées :
```bash
docker images
```

### Exécution de l'Environnement

Une fois les images chargées dans le cache local de Docker, vous pouvez lancer les scripts d'orchestration.

1. [cite_start]**Vérification du Compose :** Les fichiers `docker-compose.yaml` préconfigurés sont déjà réglés pour utiliser ces images locales au lieu de tenter un `build`[cite: 16, 19, 53].
2. [cite_start]**Lancement :** Utilisez les scripts Bash fournis pour automatiser le câblage Open vSwitch et la configuration réseau[cite: 56, 70]:

Voici un exemple d'exécution :
```bash
chmod +x setup_solution2Lab.sh
./setup_solution2Lab.sh
```



