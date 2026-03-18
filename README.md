# monitoring-homelab

Mise en place d'un service de monitoring sur mon homelab pour voir en temps réel ce qu'il se passe sur mes machines.
J'ai choisi Node Exporter, Prometheus et Grafana plutôt que des alternatives plus simples comme Beszel, pour me rapprocher de ce qu'on utilise vraiment en entreprise.

**ARCHITECTURE**

Node Exporter (:9100) ──┐
                         ├──► Prometheus (:9090) ──► Grafana (:3000)
windows_exporter (:9182)─┘

Node Exporter capture les métriques de mon Raspberry Pi 5, ensuite Prometheus vient les scraper toutes les 15 secondes et j'utilise Grafana pour que ce soit beaucoup plus lisible.

**MES CHOIX TECHNIQUES**

Pour un peu plus de sécurité j'ai créé des utilisateurs dédiés sans home ni shell, comme ça rien n'est relié à l'utilisateur que j'utilise et cela réduit la zone de possible attaque.
J'ai par la suite créé de nouvelles règles dans mon UFW pour autoriser la connexion uniquement à mon réseau LAN ainsi qu'à mon VPN.

**INSTALLATION**

`cd /tmp && wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-arm64.tar.gz`
Je me suis déplaceé dans /tmp (un dossier temporaire vidé au redémarrage),
et ensuite je télécharge l'archive via Github, J'ai utilisé la version arm64 car le Raspberry pi 5 tourne sur cette architecture. Le "&&" fait que wget ne se lance que si le cd a réussi.

`tar xvf node_exporter-1.8.2.linux-arm64.tar.gz`
on extrait l'archive tout en restant dans le dossier /tmp .
Avec tar xvf     x= extrait      v=verbose (montre tout ce qu'il extrait)      f=file (pour specifier le fichier archive à traiter)

`sudo mv node_exporter-1.8.2.linux-arm64/node_exporter /usr/local/bin/`
Je deplace le binaire "node_exporter" dans mon dossier /usr/local/bin/ ( endroit ou j'installe tout mes outils manuellement ).

`sudo useradd --no-create-home --shell /bin/false node_exporter`
Je créé un utilisateur dédié pour node_exporter sans shell et sans home comme ca personne ne pourra se connecter a cette utilisateur la.

`sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter`
Avec la commande chown je modifie le propriétaire et le groupe du binaire de node_exporter

`sudo nano /etc/systemd/system/node_exporter.service`
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
Ensuite `sudo systemctl enable node_exporter` je créé un lien Symlink pour qu'il ce lance en meme temps que mon raspberry pi 5, comme ca a chaque redemarage il ce lancera automatiquement sans la commande `sudo systemctl start node_exporter`
et ensuite je le lance avec la commande `sudo systemctl start node_exporter`.
