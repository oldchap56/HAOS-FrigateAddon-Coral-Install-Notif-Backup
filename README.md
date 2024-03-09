# HAOS-Frigate-addon-Install-Notif-Backup
## Introduction
Une grande documentation existe pour installer Frigate dans un Container LSX, mais je n'en ai pas trouvé pour installer une suite Frigate add-on, avec Back-up et notification sur smartphone
Ayant pas mal peiné et échangé sur ces sujets, je pense intéressant de faire  profiter les autres de ce qui marche chez moi. 
Ceci est d'ailleurs une Version 1 de la documentation, car il y a pas mal de points à améliorer. Je les intègrerai au fil des modifications de ma configuration.
## Prérequis
### Il est condidéré que vous avez pour 
- installé Proxmox
- intallé HA
- intallé les add-ons suivants : HAOS, Mosquitto browser
- mis en place votre accès extérieur : pour ma part avec les add-ons DuckDNS et NGINX  
- testé vos caméras avec le protocole rtsp ou onvif (ou autre mais je n'ai pas testé)
### Déroulé du tutoriel :
- Branchement de Coral dans la VM HAOS
- Installation de Frigate add-on
- Intégration de Frigate (hé oui c'est différent, j'ai eu du mal à comprendre ça)
- Construction de la Configuration Frigate
- Téléchargement de l’add-on “rclone backup” qui va servir à exporter les photos et vidéos de personnes dans google-drive
- Construction d’une Automatisation pour relancer l’add-on ‘Rclone backup) dès qu’un évènement (détection) est généré par Frigate, pour avoir des enregistrements sauvegardés dans Google drive assez vite après l’action
- Configuration des notifications à l’aide du blueprint de SgtBatten
- Création d’une Automatisation pour envoyer des photos et textes par Telegram_bot
- Ajout dans Alarmo de 2 actions liées aux alarmes : 
  - activation des détections caméras si Alarme armée (sert à faire des essais quand on ve
  - désactivation des détections caméras si Alarme désarmée (sert à faire des essais quand on veut)
## Branchement de Coral dans la VM HAOS
Il y a de multiples tutoriels sur "Comment faire en sorte d'accéder à Coral depuis Frigate", toutes basées sur l'intégration à partir du numéro du port USB.
Ici je vous montre une méthode qui habituellement n'est pas usitée, car j'ai préféré utiliser faire une intégration à partir du nom ou de l'ID du composant Coral, en particulier car je ne suis pas un pro de Linux donc j'avais des difficultés à repérer le port USB, l'intégrer à travers l'interface Proxmox, etc.
Je n'ai pas mis de photos car je pense que vous avez déjà du intégrer des dongle USB à votre Home Assistant OS

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

