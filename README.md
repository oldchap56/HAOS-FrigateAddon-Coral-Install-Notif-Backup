# HAOS-Frigate-addon-Install-Notif-Backup
## Introduction
Une grande documentation existe pour installer Frigate dans un Container LSX, mais je n'en ai pas trouvé pour installer une suite Frigate add-on, avec Back-up et notification sur smartphone
Ayant pas mal peiné et échangé sur ces sujets, je pense intéressant de faire  profiter les autres de ce qui marche chez moi. 
Ceci est d'ailleurs une Version 1 de la documentation, car il y a pas mal de points à améliorer. Je les intègrerai au fil des modifications de ma configuration.
## Prérequis
### Il est condidéré que vous avez pour 
- installé Proxmox
- intallé HAOS
- intallé les add-ons suivants : HACS, Mosquitto browser
- mis en place votre accès extérieur : pour ma part avec les add-ons DuckDNS et NGINX  
- testé vos caméras avec le protocole rtsp ou onvif (ou autre mais je n'ai pas testé)
### Déroulé du tutoriel :
- Branchement de Coral dans la VM HAOS
- Installation de Frigate add-on
- Construction de la Configuration Frigate
- Intégration de Frigate (hé oui c'est différent, j'ai eu du mal à comprendre ça)
- Téléchargement de l’add-on “rclone backup” qui va servir à exporter les photos et vidéos de personnes dans google-drive
- Construction d’une Automatisation pour relancer l’add-on ‘Rclone backup) dès qu’un évènement (détection) est généré par Frigate, pour avoir des enregistrements sauvegardés dans Google drive assez vite après l’action
- Configuration des notifications à l’aide du blueprint de SgtBatten
- Création d’une Automatisation pour envoyer des photos et textes par Telegram_bot
- Ajout dans Alarmo de 2 actions liées aux alarmes : 
  - activation des détections caméras si Alarme armée (sert à faire des essais quand on ve
  - désactivation des détections caméras si Alarme désarmée (sert à faire des essais quand on veut)
## 1. Branchement de Coral dans la VM HAOS
Il y a de multiples tutoriels sur "Comment faire en sorte d'accéder à Coral depuis Frigate", toutes basées sur l'intégration à partir du numéro du port USB.
Ici je vous montre une méthode qui habituellement n'est pas usitée, car j'ai préféré utiliser faire une intégration à partir du nom ou de l'ID du composant Coral, en particulier car je ne suis pas un pro de Linux donc j'avais des difficultés à repérer le port USB, l'intégrer à travers l'interface Proxmox, etc.
Je n'ai  mis qu'une photo car je pense que vous avez déjà du intégrer des dongle USB à votre Home Assistant OS

Donc, après avoir branché votre Clé CORAL (avec un cable USB et non directement sur votre ordi (pour ma part un NUC), il faut :
- aller dans l'interface Proxmox,
- cliquer sur votre VM HAOS
- aller sur la ligne "Matériel"
- cliquer en haut sur "Ajouter", puis "Ajouter périphérique USB"
- choisir dans le menu déroulant "Utiliser les identifiants USB du périphérique et du fabriquant"
- enfin choisir le péréphérique correspondant à Coral.
### ATTENTION : il faudra faire 2 fois cette manipulation car le périphérique Coral change de nom.
- la première fois pour moi il était vu comme constructeur GOOGLE avec un id
- après quelques tests de Frigate il avait disparu de la config et il a donc fallu le réinstaller, cette fois ci il n'a pas de nom constructeur il faut l'identifier avec son id (qui avait aussi changé)
- après cette seconde intégration il est stable chez moi et reconnu dans Frigate (on verra plus loin comment en être sûr).

    ![Coral dans VM HAOS](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/219aca98-1129-495c-9c5b-67719a8e3805)

## 2. Installation de Frigate add-on

- Dans votre Home Assistant, allez dans Paramètres, Modules complémentaires, Boutique des modules, puis cliquez en haut à droire pour ajouter un nouveau répertoire d'Add-ons et ajoutez ce lien https://github.com/blakeblackshear/frigate à la liste.
- Redémarrez Home Assistant
- Allez dans Paramétres, Modules complémentaires, Boutique des modules complémentaires (en bas pour moi), recherchez Frigate et cliquez sur Frigate, suivez les indications en indiquant que vous le faîtes tourné sur HAOS (par défaut le nom sera 
- Revenez dans Paramètres, Modules complémentaires et cochez "Afficher dans la barre latérale"
- A partir de ce moment vous pourrez configurer Fritage.

## Construction de la Configuration Frigate
Puisque vous avez Frigate dans la barre latérale (à gauche pour moi), c'est là que nous allons configurer Frigate.
Pour cela je vous joins mon fichier de configuration avec ccommentaires.

### Configutation Frigate add-on
mqtt:
  enabled: true
  host: 192.168.1.53
  port: 1883
  client_id: frigate_client
  topic_prefix: frigate
  user: mqtt-login
  password: mqtt-pass
  stats_interval: 15

detectors:
  coral:
    type: edgetpu
    device: usb

birdseye:
  enabled: false
rtmp:
  enabled: false

# Optional: Snapshot configuration  
snapshots:
  enabled: True
  # Optional: save a clean PNG copy of the snapshot image (default: shown below)
  clean_copy: True
  # Optional: print a timestamp on the snapshots (default: shown below)
  timestamp: False
  # Optional: draw bounding box on the snapshots (default: shown below)
  bounding_box: True
  # Optional: crop the snapshot (default: shown below)
  crop: False
  # Optional: Camera override for retention settings (default: global values)
  retain:
    # Required: Default retention days (default: shown below)
    default: 3
    # Optional: Per object retention days
    objects:
      person: 3

# Optional: Record configuration
# NOTE: Can be overridden at the camera level
record:
  enabled: True
  # Optional: Number of minutes to wait between cleanup runs (default: shown below)
  # This can be used to reduce the frequency of deleting recording segments from disk if you want to minimize i/o
  expire_interval: 60
  # Optional: Retention settings for recording
  retain:
    # Optional: Number of days to retain recordings regardless of events (default: shown below)
    # NOTE: This should be set to 0 and retention should be defined in events section below
    #       if you only want to retain recordings of events.
    days: 0
    # Optional: Mode for retention. Available options are: all, motion, and active_objects
    #   all - save all recording segments regardless of activity
    #   motion - save all recordings segments with any detected motion
    #   active_objects - save all recording segments with active/moving objects
    # NOTE: this mode only applies when the days setting above is greater than 0
    #mode: active_objects
  # Optional: Event recording settings
  events:
    # Optional: Number of seconds before the event to include (default: shown below)
    pre_capture: 5
    # Optional: Number of seconds after the event to include (default: shown below)
    post_capture: 5
    # Optional: Objects to save recordings for. (default: all tracked objects)
    objects:
      - person
    # Optional: Retention settings for recordings of events
    retain:
      # Required: Default retention days (default: shown below)
      default: 3
      # Optional: Mode for retention. (default: shown below)
      #   all - save all recording segments for events regardless of activity
      #   motion - save all recordings segments for events with any detected motion
      #   active_objects - save all recording segments for event with active/moving objects
      #
      # NOTE: If the retain mode for the camera is more restrictive than the mode configured
      #       here, the segments will already be gone by the time this mode is applied.
      #       For example, if the camera retain mode is "motion", the segments without motion are
      #       never stored, so setting the mode to "all" here won't bring them back.
      mode: active_objects
      # Optional: Per object retention days
      objects:
        person: 3


cameras:
  # Tapo C310 interne
  frigate1:
    ffmpeg:
      inputs:
        # Haute résolution
        - path: rtsp://adminc310:Ccavokk0!1@192.168.1.32:554/stream1
          input_args: preset-rtsp-restream
          roles:
            - record
        # Basse résolution
        - path: rtsp://adminc310:Ccavokk0!1@192.168.1.32:554/stream2
          input_args: preset-rtsp-restream
          roles:
            - detect
    detect:
      width: 1280 #width of the frame for the input with the detect role
      height: 720 #same for height
      fps: 5 #Recommended value of 5. Ideally, try and reduce your FPS on the camera.
      enabled: True

    objects:
      track:
        - person
#        - dog
#        - cat
#        - bird
#        - mouse
      filters:
        person:
          # Optional: minimum score for the object to initiate tracking (default: shown below)
          min_score: 0.7
          # Optional: minimum decimal percentage for tracked object's computed score to be considered a true positive (default: shown below)
          threshold: 0.8

 # Tapo 210 interne
  frigate2:
    ffmpeg:
      inputs:
       # Haute résolution
         - path: rtsp://adminc210:Ccavokk0!1@192.168.1.31:554/stream1
           input_args: preset-rtsp-restream
           roles:
            - record
        # Basse résolution
         - path: rtsp://adminc210:Ccavokk0!1@192.168.1.31:554/stream2
           input_args: preset-rtsp-restream
           roles:
            - detect
    detect:
      width: 1280 #width of the frame for the input with the detect role
      height: 720 #idem for the height
      fps: 5 #Recommended value of 5. Ideally, try and reduce your FPS on the camera.
      enabled: True

    objects:
      track:
        - person
#        - dog
#        - cat
#        - bird
#        - mouse
      filters:
        person:
          # Optional: minimum score for the object to initiate tracking (default: shown below)
          min_score: 0.7
          # Optional: minimum decimal percentage for tracked object's computed score to be considered a true positive (default: shown below)
          threshold: 0.8

# Vansview 
  frigate3:
    ffmpeg:
      inputs:
        - path: rtsp://HA:Wcavokkw!1@192.168.1.30:554/live/ch1
          input_args: preset-rtsp-restream
          roles:
            - detect
    detect:
#      width: 1280 #width of the frame for the input with the detect role
#      height: 720 #height of the frame for the input with the detect role (default: shown below)
      fps: 5 #Recommended value of 5. Ideally, try and reduce your FPS on the camera.
      enabled: True

    objects:
      track:
        - person
#        - dog
#        - cat
#        - bird
#        - mouse
      filters:
        person:
          # Optional: minimum score for the object to initiate tracking (default: shown below)
          min_score: 0.7
          # Optional: minimum decimal percentage for tracked object's computed score to be considered a true positive (default: shown below)
          threshold: 0.8

go2rtc:
  streams:
    back:
      - ffmpeg:rtsp://adminc210:Ccavokk0!1@192.168.1.31:554/stream2
      - ffmpeg:rtsp://adminc310:Ccavokk0!1@192.168.1.32:554/stream2
      - ffmpeg:rtsp://HA:Wcavokkw!1@192.168.1.30:554/live/ch1

  
## 3. Intégration de Frigate

https://github.com/blakeblackshear/frigate-hass-integration
