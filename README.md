# monitoring-homelab

Mise en place d'un service de monitoring sur mon homelab pour voir en temps réel ce qu'il se passe sur mes machines.
J'ai choisi Node Exporter, Prometheus et Grafana plutôt que des alternatives plus simples comme Beszel, pour me rapprocher de ce qu'on utilise vraiment en entreprise.

**ARCHITECTURE**
```
Node Exporter (:9100) ──┐
                         ├──► Prometheus (:9090) ──► Grafana (:3000)
windows_exporter (:9182)─┘
```
Node Exporter capture les métriques de mon Raspberry Pi 5, ensuite Prometheus vient les scraper toutes les 15 secondes et j'utilise Grafana pour que ce soit beaucoup plus lisible.

**MES CHOIX TECHNIQUES**

Pour un peu plus de sécurité j'ai créé des utilisateurs dédiés sans home ni shell, comme ça rien n'est relié à l'utilisateur que j'utilise et cela réduit la zone de possible attaque.
J'ai par la suite créé de nouvelles règles dans mon UFW pour autoriser la connexion uniquement à mon réseau LAN ainsi qu'à mon VPN.

## INSTALLATION

### Node Exporter

```ini
cd /tmp && wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-arm64.tar.gz
```
Je me suis déplaceé dans /tmp (un dossier temporaire vidé au redémarrage),
et ensuite je télécharge l'archive via Github, J'ai utilisé la version arm64 car le Raspberry pi 5 tourne sur cette architecture. Le "&&" fait que wget ne se lance que si le cd a réussi.

```ini
tar xvf node_exporter-1.8.2.linux-arm64.tar.gz
```
on extrait l'archive tout en restant dans le dossier `/tmp` .
Avec tar xvf     x= extrait      v=verbose (montre tout ce qu'il extrait)      f=file (pour specifier le fichier archive à traiter)

```ini
sudo mv node_exporter-1.8.2.linux-arm64/node_exporter /usr/local/bin/
```
Je deplace le binaire `node_exporter` dans mon dossier `/usr/local/bin/` ( endroit ou j'installe tout mes outils manuellement ).

```ini
sudo useradd --no-create-home --shell /bin/false node_exporter
```
Je créé un utilisateur dédié pour node_exporter sans shell et sans home comme ca personne ne pourra se connecter a cette utilisateur la.

```ini
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```
Avec la commande chown je modifie le propriétaire et le groupe du binaire de node_exporter

```ini
sudo nano /etc/systemd/system/node_exporter.service
```
Je crée le fichier de service dans /etc/systemd/system/  C'est le dossier où systemd va chercher les services configurés manuellement par l'admin. Sans ce fichier, systemd ne saurait pas comment lancer Node Exporter.

Une fois le fichier crée j'insère le contenu suivant:
```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
`[Unit]` contient les informations générales du service. `Description` lui donne un nom lisible. `Wants` indique qu'on préfère que le réseau soit disponible avant de démarrer, et `After` s'assure qu'on démarre seulement une fois le réseau en "UP", utile car Node Exporter doit exposer ses métriques sur le réseau.

`[Service]` définit comment lancer le processus. `User` et `Group` indiquent que le service tourne sous l'utilisateur dédié node_exporter. `Type=simple` signifie que systemd considère le service comme démarré dès que le processus est lancé. `ExecStart` donne le chemin du binaire à exécuter.


`WantedBy=multi-user.target` c'est ce qui dit à systemd de démarrer ce service au boot, lors du démarrage normal du système. C'est ce qui fait que `systemctl enable` fonctionne.
Sans cette section, `enable` ne saurait pas à quel moment du boot lancer le service.
`[Install]` indique à quel moment du boot le service doit démarrer. `WantedBy=multi-user.target` signifie qu'il se lance lors du démarrage normal du système, c'est ce qui permet à `systemctl enable` de fonctionner.

Pour finir l'installation de Node Exporter j'effectue la commande `sudo systemctl daemon-reload`, commande qui dit à systemd de relire ses fichiers de configuration sans redémarrer. On vient de créer le fichier `node_exporter.service` ,sans cette commande, systemd ne le connaîtrait pas encore.
Ensuite `sudo systemctl enable node_exporter` je créé un lien Symlink pour qu'il ce lance en meme temps que mon raspberry pi 5, comme ça à chaque redemarage il ce lancera automatiquement sans la commande `sudo systemctl start node_exporter`
et ensuite je le lance avec la commande `sudo systemctl start node_exporter`.

### Prometheus

Par la suite je me redéplace dans mon dossier /tmp puis je telecharge Prometheus avec `wget https://github.com/prometheus/prometheus/releases/download/v2.51.0/prometheus-2.51.0.linux-arm64.tar.gz` Prometheus est un utilitaire qui permet de scrapper les données métriques capturer par Node_Exporter, et qui, grâce a cela Grafana pourra venir recuperer les données.
Comme pour Node_Exporter, j'extrais l'archive dans le dossier /tmp avec la commande `tar xvf prometheus-2.51.0.linux-arm64.tar.gz`.

Et ensuite je deplacer les fichiers dans mon dossier local `/usr/local/bin`.
```ini
sudo mv prometheus-2.51.0.linux-arm64/prometheus /usr/local/bin/
sudo mv prometheus-2.51.0.linux-arm64/promtool /usr/local/bin/
```
le binaire `prometheus` est le binaire principal, c'est lui qui permet de tourné en arrière plan, scrape les métrique de Node_Exporter toutes les 15 secondes et les stocke dans sa base de données TSDB (Time Series DataBase, une base de données optimisée pour stocker des valeurs dans le temps).
le binaire `promtool` lui permet de vérifier la syntaxe du fichier prometheus.yml.

Pour la suite je crée deux dossiers distinct : un qui ira dans `etc/prometheus/` cela servira pour stocker la configuration de Prometheus et l'autre dans `/var/lib/prometheus/` qui lui servira pour stocker les données collectées.
Je déplace le fichier de configuration dans le dossier éxpliquer plus haut 
```ini
sudo mv prometheus-2.51.0.linux-arm64/prometheus.yml /etc/prometheus/
```
Comme pour Node_Exporter je crée un utilisateur dédié sans Home ni Shell pour plus de sécurité et je change le propriétaire et le groupe de mes dossiers Prometheus pour qu'il appartienne à mon utilisateur prometheus uniquement.
```ini
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

ensuite je crée un fichier dans `/etc/systemd/system/` qui ce nommera `prometheus.service` donc 
```ini
sudo nano /etc/systemd/system/prometheus.service 
```
dans ce fichier j'insère ceci :
```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/

[Install]
WantedBy=multi-user.target
```
C'est exactement comme pour node_exporter à la difference du user et du group puis également aux deux lignes que j'ai ajouter :
`--config.file=/etc/prometheus/prometheus.yml \ indique ou ce trouve le fichr de configuration`
`--storage.tsdb.path=/var/lib/prometheus/ indique ou ce trouve le fichier ou sont stocker les métriques`

Et comme pour node exporter je termine par : 
```ini
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

Par la suite je vais modifier le fichier `prometheus.yml` pour ajouté les métriques capturé par node_exporter j'effectue la commande suivante :
```ini
sudo nano /etc/prometheus/prometheus.yml
```
J'ajoute deux nouvelles cibles dans le fichier de configuration, mon Raspberry pi 5 via Node Exporter, et mon PC Windows via windows_exporter, que j'avais préalabement installé, accessible par WireGuard : 
```ini
# Ajout de node_exporter
  - job_name: "node_exporter"
    static_configs:
      - targets: ["localhost:9100"] # IP:port de Node Exporter

# Ajout de windows_exporter
  - job_name: "Windows"
    static_configs:
      - targets: ["<IP_WIREGUARD>:9182"] # IP WireGuard du PC Windows : port windows_exporter
```
Pour verifier que mes cibles sont bien actif je vais sur l'ip de mon raspberry pi 5 via mon navigateur internet : <IP_RASPBERRY:9090>, je clique sur "Status" et "Targets" et je vois bien que mes deux cibles sont en "UP".

### Grafana

Afin de pouvoir installer via apt Grafana, il faut pour commencé ajouter la clé GPG (clé cryptographique) afin que apt puisse vérifier que les paquets viennent bien de Grafana.
Pour cela je crée d'abord un dossier dans le dossier `/etc/apt/` qui ce nommera `/keyrings/` donc: 
```ini
sudo mkdir -p /etc/apt/keyrings
```
le `-p` crée tout les dossiers intermediraires s'ils n'existent pas déjà, sans `-p`, si `/etc/apt/` n'existait pas, la commande échouerait. Avec `-p`, il crée tout ce qui manque.
Dans mon cas, dans mon OS Raspberry pi OS lite le dossier existait déjà, donc le `-p` sert surtout à éviter une erreur si le dossier `keyrings` était déjà existant.

Une fois le dossier crée, je vais télécharger la clé GPG de Grafana, que je la convertis et que je l'ecrive dans le dossier crée précédemment. Pour cela: 
```ini
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```
`wget -q -O - https://apt.grafana.com/gpg.key |` Télécharge la clé sur le site GPG officielle de grafana. le `-q` = silencieux, pas d'affichage de progression. Le `-O -` envoie le résultat dans le pipe suivant (`|`) au lieu de le sauvegarder dans un fichier.
`gpg --dearmor |` Reçois la clé GPG du pipe précédent, et comme la clé de base est en format ASCII, `apt` ne comprend pas ce format donc je la convertit en format binaire que apt peut lire avec la commande `--dearmor` et je l'envoie dans le pipe suivant (`|`).
`sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null` Une fois la clé convertie reçu, je l'écris ave la commande `tee` dans le fichier `/etc/apt/keyrings/grafana.gpg`. Je redirige l'affichage de `tee` dans `> /dev/null` pour supprimer l'affichage de la sortie.

Ensuite j'ajoute le dépôt officelle de Grafana dans la liste des dépôts de Debian:
```ini
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```
`echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" |` La commande `echo` Génère le texte, le pipe envoie le resultat et `sudo tee /etc/apt/sources.list.d/grafana.list` écrit le resultat dans le dossier `/etc/apt/sources.list.d/grafana.list`.
Pour que mon OS valide et télécharge depuis le dépôt de Grafana, il faut que je précise que j'ai bien la clé GPG. Je lui demande donc de vérifier dans `[signed-by=/etc/apt/keyrings/grafana.gpg]`.

Une fois cela éffectuer, je met a jour les paquets avec:
```ini
sudo apt update
```
J'installe maintenant grafana:
```ini
sudo apt install grafana -y
```
Je crée un lien symlink pour qu'il démarre automatiquement au démarrage de mon Raspberry pi:
```ini
sudo systemctl enable grafana-server
```
Je démarre le service:
```ini
sudo systemctl start grafana-server
```
`grafana-server` et non simplement `grafana` car c'est le nom du service systemd que Grafana utilise le binaire s'appelle grafana mais le service systemd s'appelle grafana-server.

Maintenant j'utilise l'IP de mon raspberry pour accéder a l'interface graphique de grafana : <IP_RASPBERRY:3000> (:3000, port de base de Grafana).
Une fois identifier avec les identifiants de base, je dois ajouter un connection pour que grafana récupère bien les données scrapper par prometheus, je me dirige sur "connection" et "add new connection"
Je sélectionne **Prometheus** et je renseigne l'URL <http://localhost:9090>. Je clique sur **Save & test**, Grafana confirme que la connexion avec Prometheus est étable.
j'ai également ajouté mon pc Windows 11 donc pour cela j'ai dû ouvrir le port 9182 dans le pare-feu Windows pour autorisé la connexion. Cela fait je refait la même manipulation à la difference que j'ai dû mettre l'IP de mon pc : <IP_WireGuard:9182>, Wireguard car mon pc est toujours connécter au VPN du Raspberry.
Pour afficher les résultats, je dois importé un Dashboard, il en existe énormément car nous pouvons le personnalisé comme bon nous semble. J'ai donc sélectionné un Dashboard crée par la communauté.
Pour cela je clique sur **Dashboard** ensuite, je haut a droite **new** et **import**
Je suis ensuite parti sur https://grafana.com/grafana/dashboards/ pour trouver une interface qui me satisfaisait. Pour mon Raspberry Pi j'ai selectionner l'ID : 1860 (Node Exporter Full) et pour mon Windows j'ai selectionner l'ID : 10467 (Windows monitoring Dashboard)
