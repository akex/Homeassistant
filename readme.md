# Guide : Installation de Frigate sur Beelink S12 (Intel N100)
## Proxmox LXC non privilégié + OpenVINO (iGPU)

Ce guide résume la configuration complète pour faire tourner **Frigate NVR** dans un conteneur **LXC non privilégié** sur **Proxmox VE**, en utilisant l'iGPU de l'**Intel N100** pour l'accélération FFmpeg et la détection d'objets avec **OpenVINO**.

---

## 1. Configuration du Passthrough iGPU sur l'hôte Proxmox

### Étape 1.1 : Identifier les IDs du GPU
Connectez-vous en SSH sur votre hôte Proxmox et exécutez :
```bash
ls -l /dev/dri
```
Repérez les numéros majeurs/mineurs ainsi que le groupe (souvent `render`). Cherchez ensuite les GIDs associés sur l'hôte :
```bash
cat /etc/group | grep -E "video|render"
```
*Exemple de retour : `video:x:44:` et `render:x:104:` (Notez bien vos chiffres 44 et 104).*

### Étape 1.2 : Configurer le fichier du conteneur LXC
Ouvrez le fichier de configuration de votre LXC (remplacez `100` par l'ID de votre conteneur) :
```bash
nano /etc/pve/lxc/100.conf
```
Ajoutez les lignes suivantes tout en bas (en adaptant `44` et `104` avec vos GIDs obtenus à l'étape précédente) :
```text
# Autoriser l'accès aux caméras/GPU
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm

# Monter les composants graphiques dans le conteneur
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file

# Mapping UID/GID pour LXC non privilégié
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 44
lxc.idmap: g 44 44 1
lxc.idmap: g 45 100045 59
lxc.idmap: g 104 104 1
lxc.idmap: g 105 101105 64432
```

### Étape 1.3 : Autoriser le mapping sur l'hôte
Modifiez le fichier d'autorisations des groupes subgid de Proxmox :
```bash
nano /etc/subgid
```
Ajoutez ces deux lignes à la fin (adaptez si vos GIDs diffèrent de 44 et 104) :
```text
root:44:1
root:104:1
```

---

## 2. Déploiement de Frigate (Docker-Compose)

Une fois le LXC démarré, installez-y Docker et créez l'architecture suivante.

### `docker-compose.yml`
```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true # Recommandé pour l'accès fluide à /dev/dri dans le LXC
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
      - "8554:8554" # Flux RTSP (Go2RTC)
      - "8555:8555" # WebRTC
```

### `config/config.yml`
```yaml
mqtt:
  enabled: false # Activez et configurez pour la liaison Home Assistant

# Détecteur OpenVINO exploitant l'iGPU de l'Intel N100
detectors:
  ov:
    type: openvino
    device: GPU

# Accélération matérielle FFmpeg globale (Intel QuickSync QSV)
ffmpeg:
  hwaccel_args: preset-intel-qsv-h264 # Remplacez par preset-intel-qsv-h265 si caméras en H.265

cameras:
  camera_exemple:
    ffmpeg:
      inputs:
        # Flux secondaire de basse résolution (ex: 720p ou moins) pour la détection
        - path: rtsp://user:password@192.168.1.50:554/stream2
          roles:
            - detect
        # Flux principal haute résolution pour les enregistrements
        - path: rtsp://user:password@192.168.1.50:554/stream1
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

## 💡 Conseils de performance
1. **Premier démarrage :** Le premier lancement prendra 1 à 3 minutes. OpenVINO doit compiler le modèle YOLOx spécifiquement pour l'iGPU Intel. C'est un comportement normal.
2. **Substreams :** Utilisez impérativement des flux basse résolution pour le rôle `detect` afin de maintenir l'utilisation du processeur et de l'iGPU au plus bas.
