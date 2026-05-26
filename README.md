# Docker Netlab - Sorbonne Université

## Présentation du projet
Ce projet, réalisé dans le cadre du Master 1 (Parcours RES) à Sorbonne Université, propose une solution légère et isolée pour l'expérimentation réseau. L'objectif est de remplacer les infrastructures matérielles lourdes (routeurs Cisco, switchs physiques) par un environnement conteneurisé performant basé sur **Docker**, **Open vSwitch (OVS)** et **FRRouting (FRR)**.

Le projet permet de simuler des topologies allant de la simple commutation de niveau 2 (L2) au routage inter-VLAN de niveau 3 (L3), tout en intégrant des fonctionnalités avancées d'ingénierie de trafic comme le **Port Mirroring (SPAN)**, le **Routage Dynamique (RIPv2)**, et l'émulation de contraintes réseau.

## Nouveauté : Stack d'observabilité
La Topologie 3 intègre désormais une architecture d'observabilité complète permettant de monitorer les performances système et réseau en temps réel, essentielle pour valider la stabilité du routage dynamique :
* **cAdvisor :** Collecte les métriques système (CPU, RAM, Réseau) de chaque conteneur.
* **Prometheus :** Base de données temporelle qui scrape les métriques à intervalle régulier.
* **Grafana :** Interface de visualisation permettant de corréler les charges système avec le trafic généré par les tests (ping, iPerf).

## Structure du dépôt
```text
.
├── Topologies/
│   ├── Topology1_InternetRequired/    # Topologie L2 (Switching)
│   ├── Topology2_InternetRequired/    # Topologie L3 (Routing Statique)
│   ├── Topology2_preconfigured/       # Topologie L3 préconfigurée (hors-ligne)
│   └── Topology3_preconfigured/       # Topologie L3 avancée + Stack Monitoring
│       ├── Dockerfiles_List/          # Images locales préconfigurées
│       └── docker-compose.yaml        # Stack complète incluant Prometheus/Grafana
├── Labs/                              # Scénarios applicatifs (FTP, SSH, TFTP...)
├── captures_trafic/                   # Répertoire de stockage des captures .pcap
└── setup_solution.sh                  # Scripts d'orchestration
```

## Utilisation des environnements préconfigurés
Le projet utilise deux approches pour assurer la portabilité en salle de TP sans accès Internet :

### 1. Construction locale (nécessite Internet)
Si vous disposez d'un accès, construisez les images localement :
```bash
# Pour la stack monitoring (dans Dockerfiles_List/)
docker build -t prometheus-img:latest ./prometheus
docker build -t grafana-img:latest ./grafana
docker build -t cadvisor-img:latest ./cadvisor
```

### 2. Flux de déploiement hors-ligne (save & load)
Pour les machines isolées, utilisez l'archive `images_netlab.tar` à placer à la racine du projet (`PRES/`) :
```bash
docker load -i images_netlab.tar
```

## Exécution de l'environnement
Une fois les images chargées, lancez le script d'automatisation. Le script gère désormais le cycle de vie complet, y compris le nettoyage des conteneurs de monitoring :
```bash
chmod +x setup_solution3Lab.sh
./setup_solution3Lab.sh
```

## Visualisation du monitoring
Une fois l'infrastructure lancée :

1. Accédez à Grafana via `http://localhost:3000` (admin/admin).
2. Utilisez la source de données `http://prometheus:9090`.
3. Dans l'onglet **Explore**, utilisez la requête suivante pour visualiser les consommations mémoire de tous les conteneurs :

```promql
container_memory_usage_bytes{name=~".+"}
```

## Visualisation du trafic capturé
Le test génère un fichier `.pcap` dans `/home/debian/PRES/captures_trafic/trafic_topo3/`.

* **Wireshark :** Filtrez par `rip` pour observer les mises à jour de routage, ou `icmp` pour vérifier la connectivité entre le Client et le Serveur.
* **Vérification routage :** `docker exec router1 ip route` confirme la convergence RIPv2.

## Conclusion
Ce projet démontre la maîtrise d'une architecture SDN résiliente et instrumentée. L'intégration de la stack d'observabilité "Full-Stack" garantit une visibilité totale sur le comportement dynamique des protocoles de routage dans un environnement conteneurisé, offrant une solution robuste pour l'enseignement et l'expérimentation réseau.
