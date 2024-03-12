# HAOS-Frigate-addon-Install-Notif-Backup
## AVERTISSEMENT
### Ce tutoriel est en version "A valider" et n'est pas forcément utilisable en l'état ! J'en ferai une version 1 "validée" dés que possible.

## Introduction
Une grande documentation existe pour installer Frigate dans un Container LSX, mais je n'en ai pas trouvé pour installer une suite Frigate add-on, avec Back-up et notification sur smartphone.

Ayant pas mal peiné et échangé sur ces sujets, je pense intéressant de faire  profiter les autres de ce qui marche chez moi. 
Ceci est d'ailleurs une Version 0 de la documentation, car il y a pas mal de points à améliorer. Je les intègrerai au fil des modifications de ma configuration.
## Prérequis
### Il est considéré que vous avez déjà : 
- installé Proxmox
- intallé HAOS
- intallé les add-ons suivants : HACS, Mosquitto browser, Alarmo (optionnel)
- mis en place votre accès extérieur : pour ma part avec les add-ons DuckDNS et NGINX  
- testé vos caméras avec le protocole rtsp ou onvif (ou autre mais je n'ai pas testé)
- avoir configuré un bot Telegram dans HAOS.

### Déroulé du tutoriel :
- Branchement de Coral dans la VM HAOS
- Installation de Frigate add-on
- Configuration Frigate
- Intégration de Frigate (hé oui c'est différent, j'ai eu du mal à comprendre ça)
- Téléchargement de l’add-on “rclone backup” qui va servir à exporter les photos et vidéos de personnes dans google-drive
- Configuration de l'addon "rclone backup"
- Construction d’une Automatisation pour relancer l’add-on "Rclone backup" dès qu’un évènement (détection) est généré par Frigate, pour avoir des enregistrements sauvegardés dans Google drive assez vite (15 secondes) après l’intrusion
- Configuration des notifications à l’aide du blueprint de SgtBatten
- Création d’une Automatisation pour envoyer des photos et textes par Telegram_bot
- Ajout dans Alarmo (ou construction directes par Automatisation) de 2 actions liées aux alarmes : 
  - activation des détections caméras si Alarme armée (sert aussi à faire des essais quand on veut)
  - désactivation des détections caméras si Alarme désarmée (sert aussi à faire des essais quand on veut)
## 1. Branchement de Coral dans la VM HAOS
Il y a de multiples tutoriels sur "Comment faire en sorte d'accéder à Coral depuis Frigate", tous basés sur l'intégration à partir du numéro du port USB.
Ici je vous montre une méthode qui habituellement n'est pas usitée, car j'ai préféré faire une intégration à partir du nom ou de l'ID du composant Coral, en particulier car je ne suis pas un pro de Linux donc j'avais des difficultés à repérer le port USB, l'intégrer à travers l'interface Proxmox, etc.
Je n'ai  mis qu'une photo car je pense que vous avez déjà dû intégrer des dongle USB à votre Home Assistant OS

Donc, après avoir branché votre Clé CORAL - avec un cable USB et non directement sur votre ordi (pour ma part un NUC), il faut :
- aller dans l'interface Proxmox,
- cliquer sur votre VM HAOS
- aller sur la ligne "Matériel"
- cliquer en haut sur "Ajouter", puis "Ajouter périphérique USB"
- choisir dans le menu déroulant "Utiliser les identifiants USB du périphérique et du fabriquant"
- enfin choisir le péréphérique correspondant à Coral.
### ATTENTION : il faudra faire 2 fois cette manipulation car le périphérique Coral change de nom.
- la première fois pour moi il était vu comme constructeur GOOGLE avec un id
- après quelques tests de Frigate il avait disparu de la config et il a donc fallu le réinstaller, cette fois ci il n'a pas de nom constructeur il faut l'identifier avec son id (qui a aussi changé)
- après cette seconde intégration il est stable chez moi et reconnu dans Frigate (on verra plus loin comment en être sûr).

## 2. Installation de Frigate add-on

- Dans votre Home Assistant, allez dans Paramètres, Modules complémentaires, Boutique des modules, puis cliquez en haut à droire pour ajouter un nouveau répertoire d'Add-ons et ajoutez ce lien https://github.com/blakeblackshear/frigate à la liste.
- Redémarrez Home Assistant
- Allez dans Paramétres, Modules complémentaires, Boutique des modules complémentaires (en bas pour moi), recherchez Frigate et cliquez sur Frigate, suivez les indications en indiquant que vous le faîtes tourner sur HAOS.

- Revenez dans Paramètres, Modules complémentaires et cochez "Afficher dans la barre latérale"
  
 ![Frigate Add-on](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/1ea7e5a3-8082-4348-9e0e-3d1826be2c57)

A partir de ce moment vous pourrez configurer Frigate.

## 3. Construction de la Configuration Frigate
Puisque vous avez Frigate dans la barre latérale (à gauche pour moi), c'est par là que nous allons le configurer.
Pour cela je vous joins mon fichier de configuration, il se  nomme "Config Frigate addon oldchap56" et les commentaires sont à l'intérieur. Pour avoir des détails sur le contenu je vous conseille l'excellent tutoriel dont il est tiré, rédigé par Raynox et que vous trouverez sur https://www.youtube.com/watch?v=-haxDKIOEao

En gros les options choisies permettent à chaque détection d'un humain par Frigate (fait sur le flux basse résolution) de faire un snapshot et une vidéo avec le flux haute résoution. 
Ces shapshots et vidéos sont accessibles :
- soit dans l'interface Frigate, en cliquant sur Events, on a toutes le détections classées chronologiquement,
- soit dans l'interface Médias, classées par date puis caméra puis à nouveau chronologie.
  
## 4. Intégration de Frigate
Cette intégration permet d'utiliser les capteurs, évènements etc. de Frigate qui discute en mqtt avec le reste du monde (HAOS, les caméras etc.). C'et ce qui permet de créer des Automatisations à partir de ce qui se passe à l'intérieur de Frigate.
Pour l'intégration, il faut aller dans HACS (barre de gauche), Intégrations, cliquer sur Dépots personnalisés pour inclure le lien suivant dans la catégorie Intégration
https://github.com/blakeblackshear/frigate-hass-integration.
Ensuite, cliquer dessus pour l'installer. [A VALIDER]
Nota : on peut aussi l'intégrer directement à partir du bouton bleu dans le README du Github.

## 5. Téléchargement de l’add-on “rclone backup” 
Il va servir à exporter les photos et vidéos de personnes dans google-drive au fil de l'eau.
Ici encore il va falloir installer le lien dans les dépôts (Paramètres, Modules complémentaires, Boutiques des modules complémentaires, Trois petits  points en haut à droite, ajout de 
(https://github.com/jcwillox/hassio-rclone-backup) 
Puis redémarrage de HAOS, Boutique de modules complémentaires, Ajouter Rclone back-up.
Nota : on peut aussi l'intégrer directement à partir du bouton bleu dans le README du Github.

## 6. Configuration de l’add-on “rclone backup” 
Ma configuration de Rclone backup est particulière, j'ai dû "tordre" un peu l'addon car il fonctionne de base comme une sauvegarde à des moments fixés d'avance (jours, semaines, mois, années configurées dans un cron). 
Mon fonctionnement consiste à redémarrer Rclone back up à chaque détection et non une fois tous les jours.
Pour la configuration de Rclone back-up, ma est configuration (accessible dans le fichier "Job pour Rclone"). Elle permet de lancer le backup à chaque démarrage de l'add-on (L'automatisation décrite dans le § suivant permettra de le relancer).
Description du job :
- pas de mention du moment de la sauvegarde (ligne effacé) pour permettre un backup à chaque démarrage de l'addon
- la commande est copyto qui copie à chaque lancement les fichiers vers Google drive (donc je gère à la main les fichiers dns google drive pour la destruction par exemple). Si vous souhaitez synchroniser la source (HAOS) et la destination (Google Drive) qui auront toujours le même contenu, mettez à la place la commande rsynch [A VALIDER].
- source : /media/frigate est l'endroit où Frigate met les fichiers par défaut.
- destination : google-drive:frigate. frigate est le répertoire à la racine de mon google drive où je fais mes sauvebardes.
- 
### ATTENTION : il faut configurer l'accès à Google drive par rclone ! 
Pour cela il faut obtenir de Google Drive un client id / client secret (qui sont créés pr Google Drive sur son API de connection des applications), permettant a Rclone de déposer les fichiers : https://rclone.org/drive/#making-your-own-client-id
Mais, sous HAos, je n'ai pas réussi à générer le token  qui et une troisième information nécessaire pour connecter une application à google drive.
J'ai donc fait (comme Ryan72) une installation rclone sous Linux (en ligne de commande avec la commande rclone config) qui a fonctionnée tout de suite, ce qui m'a permis  afin de récupérer le token à partir des clientid/client secret déjà obtenus.
J'ai scrupuleusement copié les 3 informations (client id, client secret, token) ce qui m'a parmis de les integrer ensuite dans home assistant au niveau de la configuration d'accès sous "rclone back-up".
Si tout ça a bien marché vous pourrez voir que vous êtes connectés en cliquant sur Rclone back-up dans le panneau de gauche, en particulier en utilisant le menu de gauche Explorer et en "montant" Google Drive. Vous devez alors voir tous vos répertoires Gdrive dans le tableau. C'est la preuve que tout est bien configuré pour Rclone back-up.
### Re attention !
Au début on fonctionne vu de Google avec un mode "test" pour rclone. Je suppose que le délai de validité du token est très réduit dans ce mode. J'ai redemandé un token et ça a bien marché tout de suite .... A SUIVRE.

## 7. Construction d’une Automatisation pour relancer l’add-on "Rclone backup"
Ceci permet de relancer la copie vers Gdrive dès qu’un évènement (détection) est généré par Frigate, pour avoir des enregistrements sauvegardés dans Google drive assez vite après l’action (au cas où par exemple les intrus volent ou détruisent votre serveur HAOS).
L'automatisation est dans un fichier de ce tutoriel sous le nom "Redémarrage Rclone backup si détection humain"
Remarque :  j'ai mis 15 secondes de temporisation avant de démarrer la copie car les vidéos sauvegardées par Frigate dans HAOS sont découpées en tronçons de 10 secondes, comme ça la copie démarre avec au moins une vidéo à sauvegarder.

## 8. Configuration des notifications à l’aide du blueprint de SgtBatten
Rien de compliqué pour cela, ça vous permet de recevoir sur votre smartphone une notification en cas d'intrusion avec un snapshot et en cliquant dessus l'accès aux vidéos à travers l'application mobile de Home Assistant.
Pour cela :
- allez dans Paramètres, Automatisation, Blueprint et allez chercher l'automatisation Frigate Notifications de SgtBatten.
- créez une Automatisation à l'aide du Blueprint. J'ai mis mon fichier Yaml pour cette Automatidation dans le fichier joint "Détection intrusion & envoi Notification HA"

![Notification HA-BD](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/51406148-95b9-4608-9c1f-4dfac72a58a3)

Mais, ceci ne me suffisait pas car on ne peut a priori envoyer la Notification que sur 1 portable (je n'ai pas réussi à faire un Groupe de portables dans HA).
Donc, j'ai fait une autre Automatisation qui envoie l'information sur un bot Telegram. Ainsi tous les membres dela famille qui en font partie recevront l'alerte en cas d'intrusion.


## 9. Création d’une Automatisation pour envoyer des photos et textes par Telegram_bot
Pour faire ça il faut avoir configuré un Bot Telegram. Si ce n'est fait je vous conseille ce tutoriel Youtube de Maternix (https://www.youtube.com/watch?v=gJpnIslsLqU)
Maintenant que vous avez votre bot Telegram configuré et utilisable dans Home Assistant, il ne reste plus qu'à faire une automatisation !
La configuration yaml de cette automatisation se trouve dans le fichier joint "Notification Telegram Frigate Intrusion".

![Telegram bot-BD](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/318790c0-2a7f-4a17-a330-c30a284d9363)

## 10. Ajout dans Alarmo de 2 actions liées aux alarmes : 
Ces 2 Automatisations créées à partir d'actions créées dans Alarmo servent à activer ou non les détections selon que l'alarme est déclenchée ou non.
Elles servent aussi à faire des essais de réglage en les déclenchant manuellement au lieu de mettre l'alarme (ici exemple où on démarre les détections et les enregistrements en appuyant sur ESSAI)
![démarrage détection](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/27ca58bb-26c4-4206-86a4-d4c22a7a08c8)



Les 2 actions sont configurées dans Alarmo et le fichier exemple est "Détection démarrage action Alarmoo
 - activation des détections caméras si Alarme armée (sert à faire des essais quand on ve
  - désactivation des détections caméras si Alarme désarmée (sert à faire des essais quand on veut)


