# Guide d'Installation : Frigate NVR avec OpenVINO sur Proxmox (LXC Non Privilégié)

Ce guide détaille l'installation pas-à-pas de Frigate NVR sur un mini-PC **Beelink S12 Pro (Intel N100)** en utilisant un conteneur **LXC Debian 12 non privilégié**. Cette méthode exploite l'iGPU Intel (via **OpenVINO**) pour la détection d'objets et le décodage matériel, offrant d'excellentes performances (~10-20ms d'inférence) sans accélérateur externe Google Coral.

---

## 1. Choix et Création du LXC sur Proxmox

L'OS recommandé est **Debian 12 (Bookworm)** pour sa légèreté, sa stabilité et sa gestion native des pilotes graphiques récents de l'architecture Intel Alder Lake-N (N100).

1. Dans l'interface Proxmox, téléchargez le template officiel `debian-12-standard`.
2. Cliquez sur **Create CT** et configurez les options suivantes :
   * **Unprivileged container :** Coché (Actif).
   * **CPU :** Au moins 2 cœurs.
   * **RAM :** 2 Go de mémoire (minimum recommandé).
   * **Root Disk :** Allouez l'espace selon vos besoins de stockage temporaire (les vidéos longues seront idéalement stockées sur un NAS ou un disque externe).

---

## 2. Configuration du Passthrough iGPU (Hôte Proxmox)

Pour qu'un conteneur non privilégié accède à la carte graphique sans droits root, il faut mapper précisément les identifiants de groupe (GID) entre l'hôte et le conteneur.

### Étape A : Identifier les IDs sur l'hôte Proxmox
Connectez-vous en SSH sur votre hôte Proxmox et exécutez :
```bash
ls -l /dev/dri
```
*Exemple de retour :*
```text
crw-rw---- 1 root video  226,   0 Sep  9 12:00 card0
crw-rw---- 1 root render 226, 128 Sep  9 12:00 renderD128
```

Trouvez les GID associés aux groupes `video` et `render` de l'hôte :
```bash
cat /etc/group | grep -E "video|render"
```
*Notez les numéros obtenus (ex: `video` = 44, `render` = 104). Les étapes suivantes utilisent 44 et 104 comme exemples.*

### Étape B : Autoriser Proxmox à partager ces IDs
Modifiez le fichier système des sous-groupes sur l'hôte Proxmox :
```bash
nano /etc/subgid
```
Ajoutez ces deux lignes à la fin (adaptez 44 et 104 avec vos valeurs) :
```text
root:44:1
root:104:1
```

### Étape C : Associer l'iGPU au fichier de configuration du LXC
Ouvrez le fichier de configuration de votre conteneur (remplacez `100` par l'ID de votre LXC) :
```bash
nano /etc/pve/lxc/100.conf
```
Ajoutez l'ensemble de ces lignes tout en bas du fichier :
```text
# Autoriser l'accès aux fichiers du GPU
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm

# Monter les composants graphiques dans le conteneur
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file

# Redirection des privilèges UID/GID (Mapping)
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 44
lxc.idmap: g 44 44 1
lxc.idmap: g 45 100045 59
lxc.idmap: g 104 104 1
lxc.idmap: g 105 101105 64431
```
*Sauvegardez et quittez (`Ctrl+O` puis `Ctrl+X`). Démarrez votre LXC.*

---

## 3. Déploiement de Docker dans le LXC

Connectez-vous à la console de votre LXC Debian 12 tout juste démarré.

### Étape A : Valider l'accès au GPU
Exécutez la commande suivante à l'intérieur du conteneur :
```bash
ls -l /dev/dri
```
Si les fichiers `card0` et `renderD128` apparaissent, le passthrough est opérationnel.

### Étape B : Installer l'environnement Docker
Exécutez le script officiel pour installer Docker de manière propre :
```bash
apt update && apt install -y curl
curl -fsSL https://docker.com -o get-docker.sh
sh get-docker.sh
```

---

## 4. Fichiers de Configuration Prêts à l'Emploi

Créez un dossier dédié à Frigate (ex: `/opt/frigate`) et placez-y les deux fichiers suivants.

### `docker-compose.yml`
```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true # Indispensable dans le Docker interne pour outrepasser les conflits de droits
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:stable
    shm_size: "128mb" # À augmenter (ex: 256mb) si vous avez plus de 4 caméras
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./config:/config
      - ./storage:/media/frigate
    ports:
      - "5000:5000"
      - "8554:8554" # Flux RTSP (via go2rtc)
      - "8555:8555" # WebRTC
```

### `config.yml` (Placé dans le sous-dossier `./config`)
```yaml
mqtt:
  enabled: false # Passez à true pour l'intégration Home Assistant

# Détecteur OpenVINO configuré sur l'iGPU Intel
detectors:
  ov:
    type: openvino
    device: GPU

# Accélération matérielle globale pour les processeurs Intel récents
ffmpeg:
  hwaccel_args: preset-intel-qsv-h264 # Remplacez par preset-intel-qsv-h265 si vos caméras émettent en H.265

cameras:
  camera_exemple:
    ffmpeg:
      inputs:
        # Flux secondaire (basse résolution type 720p ou moins) dédié à la détection
        - path: rtsp://votre_utilisateur:votre_mot_de_passe@192.168.1.50:554/stream2
          roles:
            - detect
        # Flux principal (haute résolution) dédié aux enregistrements
        - path: rtsp://votre_utilisateur:votre_mot_de_passe@192.168.1.50:554/stream1
          roles:
            - record
    detect:
      enabled: true
    record:
      enabled: true
      retain:
        days: 7
```

---

## 5. Bonnes Pratiques & Conseils pour le N100

* **Flux de détection léger :** Configurez systématiquement un flux secondaire (substream) basse résolution (ex: 640x480 ou 1280x720) pour le rôle `detect`. Faire analyser des flux 2K ou 4K par OpenVINO surchargerait inutilement l'iGPU.
* **Premier démarrage :** Lors du premier lancement, Frigate compile le modèle YOLOx spécifiquement pour l'iGPU de l'Intel N100. Cette opération prend **1 à 3 minutes** pendant lesquelles l'interface peut sembler indisponible. Laissez le processus se terminer.
* **Suivi des performances :** Une fois démarré, l'onglet **System** de l'interface Frigate vous permettra de valider le temps de réponse du détecteur `ov` ainsi que la bonne prise en charge du décodage matériel.
