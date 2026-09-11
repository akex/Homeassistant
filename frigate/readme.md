# Guide de Configuration : Frigate, Frigate-Notify et Ntfy via Tailscale (Docker séparés)

Ce guide détaille la mise en place d'une architecture sécurisée à **3 stacks Docker indépendantes** sur un conteneur LXC Debian 12 (Proxmox). Les notifications de Frigate sont transmises localement à `ntfy`, qui les pousse vers votre smartphone Android via un tunnel chiffré **Tailscale**, sans aucune ouverture de port sur votre routeur.

---

## 1. Prérequis sur Proxmox (Hôte)

Pour que le conteneur Docker `tailscale` puisse créer le tunnel VPN, le LXC de Proxmox doit être autorisé à utiliser le périphérique réseau `/dev/net/tun`.

1. Éteignez votre conteneur LXC Debian 12.
2. Connectez-vous en SSH à votre **nœud Proxmox** (l'hôte physique).
3. Ouvrez le fichier de configuration du LXC (remplacez `100` par l'ID de votre LXC) :
   ```bash
   nano /etc/pve/lxc/100.conf
   ```
4. Ajoutez ces deux lignes tout en bas du fichier :
   ```text
   lxc.cgroup2.devices.allow: c 10:200 rwm
   lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
   ```
5. Enregistrez (`Ctrl+O` puis `Entrée`) et quittez (`Ctrl+X`).
6. Redémarrez le LXC.

---

## 2. Création des Réseaux Docker Externes

Afin de permettre à la stack `frigate-notify` de communiquer avec les deux autres environnements indépendants, créez deux réseaux globaux depuis le terminal de votre LXC :

```bash
docker network create net-frigate
docker network create net-notifications
```

---

## 3. Configuration des Stacks Docker

### Stack A : Tailscale & Ntfy (`docker-compose.yml`)
Le conteneur `ntfy` fusionne sa pile réseau avec celle de `tailscale` via le mode `network_mode: "service:tailscale"`. Il est ainsi uniquement accessible à travers le VPN.

```yaml
version: '3.8'

services:
  tailscale:
    image: tailscale/tailscale:stable
    container_name: ts-ntfy
    volumes:
      - ./tailscale/state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    environment:
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
    cap_add:
      - NET_ADMIN
    networks:
      - net-notifications
    restart: unless-stopped

  ntfy:
    image: binwiederhier/ntfy
    container_name: ntfy
    command: serve
    network_mode: "service:tailscale"
    environment:
      - NTFY_BASE_URL=http://ts-ntfy:80
      - NTFY_LISTEN_HTTP=:80
    volumes:
      - ./ntfy/cache:/var/cache/ntfy
      - ./ntfy/etc:/etc/ntfy
    depends_on:
      - tailscale
    restart: unless-stopped

networks:
  net-notifications:
    external: true
```

### Stack B : Frigate (`docker-compose.yml`)
Connectez votre instance Frigate existante au réseau dédié `net-frigate`.

```yaml
version: '3.8'

services:
  frigate:
    image: ghcr.io/blakeblackshear/frigate:stable
    container_name: frigate
    # ... Conservez vos configurations (volumes, ports, devices, shm_size) ...
    networks:
      - net-frigate
    restart: unless-stopped

networks:
  net-frigate:
    external: true
```

### Stack C : Frigate-Notify (`docker-compose.yml`)
Ce conteneur sert de pont en rejoignant simultanément les deux réseaux externes.

```yaml
version: '3.8'

services:
  frigate-notify:
    image: ghcr.io/0x2142/frigate-notify:latest
    container_name: frigate-notify
    # ... Conservez vos volumes de configuration ...
    networks:
      - net-frigate
      - net-notifications
    restart: unless-stopped

networks:
  net-frigate:
    external: true
  net-notifications:
    external: true
```

---

## 4. Configuration de Frigate-Notify (`config.yml`)

Modifiez le fichier de configuration de `frigate-notify` pour cibler les conteneurs par leurs noms d'alias Docker :

```yaml
frigate:
  # Trouve le conteneur sur le réseau net-frigate
  url: http://frigate:5000 

notifiers:
  mon_ntfy:
    type: ntfy
    # Trouve le conteneur Tailscale sur le réseau net-notifications
    url: http://ts-ntfy:80 
    topic: alertes_cameras
```

---

## 5. Initialisation et Connexion

1. Démarrez les trois environnements en exécutant `docker compose up -d` dans leurs dossiers respectifs.
2. Récupérez le lien d'authentification Tailscale dans les logs du conteneur réseau :
   ```bash
   docker logs ts-ntfy
   ```
3. Cliquez sur le lien généré pour lier le conteneur à votre compte Tailscale. Notez l'adresse IP Tailscale (`100.x.y.z`) attribuée à `ts-ntfy`.

---

## 6. Configuration de l'Application Android

1. Installez et activez **Tailscale** sur votre smartphone (connecté au même compte).
2. Ouvrez l'application **ntfy**, appuyez sur le bouton `+` pour ajouter un abonnement.
3. Renseignez l'adresse de votre topic en utilisant l'adresse IP Tailscale notée précédemment : `http://100.x.y.z/alertes_cameras` (ou `http://ts-ntfy/alertes_cameras` si Tailscale DNS / MagicDNS est actif).
