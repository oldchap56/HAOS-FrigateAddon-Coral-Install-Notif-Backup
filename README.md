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
- Téléchargement et configuration de l’add-on “rclone backup” qui va servir à exporter les photos et vidéos de personnes dans google-drive
- Configuration de l'addon "rclone backup"
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
- Allez dans Paramétres, Modules complémentaires, Boutique des modules complémentaires (en bas pour moi), recherchez Frigate et cliquez sur Frigate, suivez les indications en indiquant que vous le faîtes tourner sur HAOS.

![Frigate Add-on2](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/44193782-d06a-4637-b30c-4f1a011119df)


- Revenez dans Paramètres, Modules complémentaires et cochez "Afficher dans la barre latérale"
  
 ![Frigate Add-on](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/1ea7e5a3-8082-4348-9e0e-3d1826be2c57)

A partir de ce moment vous pourrez configurer Fritage.

## 3. Construction de la Configuration Frigate
Puisque vous avez Frigate dans la barre latérale (à gauche pour moi), c'est là que nous allons configurer Frigate.
Pour cela je vous joins mon fichier de configuration avec ccommentaires, il se  nomme "Config Frigate addon oldchap56" et les commentaires sont à l'intérieur. Pour avoir des détails sur le contenu je vous conseille l'excellent tutoriel dont il est tiré, rédigé par Raynox et que vous trouverez sur https://www.youtube.com/watch?v=-haxDKIOEao

En gros les options choisies permettent à chaque détection d'un humain par Frigate (fait sur le flux basse résolution) de faire un snapshot et une vidéo avec le flux haute résoution. 
Ces shapshots et vidéos sont accessibles :
- soit dans l'interface Frigate, en cliquant sur EVents, on a toutes le détections classées chronologiquement
- soit dans l'interface Médias, classées par date puis caméra puis à nouveau chronologie.
  
## 4. Intégration de Frigate
Cette intégration permet d'utiliser les capteurs, évènements etc. de Frigate qui discute en mqtt avec le reste du monde (HAOS, les caméras etc.). C'et ce qui permet de créer des Automatisations à partir de ce qui se passe à l'intérieur de Frigate.
Pour l'intégration, il faut aller dans HACS (barre de gauche), Intégrations, cliquer sur Dépots personnalisés pour inclure le lien suivant dans la catégorie Intégration
https://github.com/blakeblackshear/frigate-hass-integration.
Ensuite, cliquer dessus pour l'installer. [A VALIDER]

## 5. Téléchargement de l’add-on “rclone backup” 
Il va servir à exporter les photos et vidéos de personnes dans google-drive.
Ici encore il va falloir installer le lien dans les dépôts (Paramètres, Modules complémentaires, Boutiques des modules complémentaires, Trois petits  points en haut à droite, ajout de 
https://github.com/ViViDboarder/hassio-addon-rclone
Puis redémarrage de HAOS, Boutique de modules complémentaires, Ajouter Rclone back-up.

## 6. Configuration de l’add-on “rclone backup” 
Ma configuration de Rclone backup est particulière, j'ai dû "tordre" un peu l'addon car il fonctionne de base comme une sauvegarde à des moments fixés d'avance (jours, semaines, mois, années configurées dans un cron). 
Mon fonctionnement consiste 
Pour la configuration de Rclone back-up, voici ma configuration qui permet de lancer le backup à chaque démarrage de l'add-on.
