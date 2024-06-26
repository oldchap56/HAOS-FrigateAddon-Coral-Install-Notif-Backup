# HAOS-Frigate-addon-Install-Notif-Backup
## AVERTISSEMENT
### [For the english version please click Here go to README_English.md](README_English.md)

## Introduction
Une grande documentation existe pour installer Frigate dans un Container LSX, mais je n'en ai pas trouvé pour installer une suite Frigate add-on, avec Back-up et notification sur smartphone.

Ayant pas mal peiné et échangé sur ces sujets, je pense intéressant de faire  profiter les autres de ce qui marche chez moi. 

## Prérequis
### Il est considéré que vous avez déjà : 
- installé Proxmox
- intallé HAOS
- intallé les add-ons suivants : HACS, Mosquitto browser, Alarmo (optionnel)
- mis en place votre accès extérieur (optionnel) : pour ma part avec les add-ons DuckDNS et NGINX  
- testé vos caméras avec le protocole rtsp ou onvif (ou autre mais je n'ai pas testé)
- avoir configuré un bot Telegram dans HAOS (optionnel).

### Déroulé du tutoriel :
- Branchement de Coral dans la VM HAOS
- Installation de Frigate add-on
- Configuration Frigate
- Intégration de Frigate (hé oui c'est différent, j'ai eu du mal à comprendre ça)
- Création d’une Automatisation pour envoyer une notification avec texte, photos et vidéos par Telegram_bot
- Configuration des notifications à l’aide du blueprint de SgtBatten
- Ajout dans Alarmo (ou construction directes par Automatisation) de 2 actions liées aux alarmes : 
  - activation des détections caméras si Alarme armée (sert aussi à faire des essais quand on veut)
  - désactivation des détections caméras si Alarme désarmée (sert aussi à faire des essais quand on veut)
#### Et aussi, un essai à moitié réussi de sauvegarde sur Google-drive :
Le moitié réussi correspond à un problème de perte de droits au bout d'un certain temps - techniquement impossible d'utiliser le token de renouvellement des droits vers google-drive :
- Téléchargement de l’add-on “rclone backup” qui va servir à exporter les photos et vidéos de personnes dans google-drive
- Configuration de l'addon "rclone backup"
- Construction d’une Automatisation pour relancer l’add-on "Rclone backup" dès qu’un évènement (détection) est généré par Frigate, pour avoir des enregistrements sauvegardés dans Google drive assez vite (15 secondes) après l’intrusion.

## 1. Branchement de Coral dans la VM HAOS
### Proposition initiale
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

### Proposition plus classique (qu'on retrouve sur des tutoriels divers)
Il s'agit d'intégrer Coral à partir du numéro de port USB  .... manipulation réussie pour moi en "cassant" la méthode ci-dessus.
Tout d'abord vous devez identifier les différents dongle USB présents sur votre "serveur HA" (pour moi un NUC).
Pour cela dans Proxmox :
- revenir au niveau du noeud sur lequel se trouve votre VM HAOS,
- sélectionner le shell de ce noeud (pour moi je l'ai appelé Proxmox
- lancer la commande lsusb, ce qui vous donnera les différents ports utilisés

![Coral 1 lsusb](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/f2a430a9-2583-43cb-a7e8-454c44d914fb)
- ensuite allez sur l'interface de la VM de HAOS (pour moi la VM 102 - VM Prod), et intégrez Coral comme ci-dessus mais cette fois ci en choisissant "utilisez le port USB", ce qui donne en détail :
  - aller dans l'interface Proxmox,
  - cliquer sur votre VM HAOS
  - aller sur la ligne "Matériel"
  - cliquer en haut sur "Ajouter", puis "Ajouter périphérique USB"
  - choisir dans le menu déroulant "Utiliser le port USB"
  - enfin choisir le péréphérique correspondant à Coral, en cliquant sur la bonne ligne 'pour moi j'ai choisi la ligne 2.6 alors que le lsusb m'avait donnée usb 2 port 5 !!! J'y suis allé par déduction ayant identifié tous les autres dongle/ports USB)
  ![Coral 2 vision VM HAOS](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/b69e1a73-04cc-4dae-a8ef-49241d6e687e)

 
### !!! Dans les 2 cas d'installation vous pouvez passer à l'installation de Frigate add-on, dans le § suivant.

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

De même pour vérifier que Coral a bien été reconnu par Frigate, vous pouvez :
- cliquer sur Frigate dans la barre de gauche,
- puis aller dans l'onglet "System",
- et là vous pourrez vérifier que Coral fonctionne bien !!

![Interface Frigate pour voir Coral](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/3f1e4fcc-942f-4fc0-8709-04150a5ef993)

  
## 4. Intégration de Frigate
Cette intégration est OBLIGATOIRE et permet d'utiliser les capteurs, évènements etc. de Frigate qui discute en mqtt avec le reste du monde (HAOS, les caméras etc.). C'et ce qui permet de créer des Automatisations à partir de ce qui se passe à l'intérieur de Frigate.
Pour l'intégration, il faut aller dans HACS (barre de gauche), Intégrations, cliquer sur Dépots personnalisés pour inclure le lien suivant dans la catégorie Intégration
https://github.com/blakeblackshear/frigate-hass-integration.
Ensuite, cliquer dessus pour l'installer.
Nota : on peut aussi l'intégrer directement à partir du bouton bleu dans le README du Github.

## 5. Configuration des notifications à l’aide du blueprint de SgtBatten
Rien de compliqué pour cela, ça vous permet de recevoir sur votre smartphone une notification en cas d'intrusion avec un snapshot et en cliquant dessus l'accès aux vidéos à travers l'application mobile de Home Assistant.
Pour cela :
- allez dans Paramètres, Automatisation, Blueprint et allez chercher l'automatisation Frigate Notifications de SgtBatten.
- créez une Automatisation à l'aide du Blueprint. J'ai mis mon fichier Yaml pour cette Automatidation dans le fichier joint "Détection intrusion & envoi Notification HA"

![Notification HA-BD](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/51406148-95b9-4608-9c1f-4dfac72a58a3)

Ici la notification par HAOS sur mon portable.

Mais, ceci ne me suffisait pas car on ne peut a priori envoyer la Notification que sur 1 portable (je n'ai pas réussi à faire un Groupe de portables dans HA).
Donc, j'ai fait une autre Automatisation qui envoie l'information sur un bot Telegram (VOIR §8 suivant). Ainsi tous les membres dela famille qui en font partie recevront l'alerte en cas d'intrusion.


## 6. Création d’une Automatisation pour envoyer une notification avec texte, photos et vidéos par Telegram_bot (optionnel)
Pour faire ça il faut avoir configuré un Bot Telegram. Si ce n'est fait je vous conseille ce tutoriel Youtube de Maternix (https://www.youtube.com/watch?v=gJpnIslsLqU)

Maintenant que vous avez votre bot Telegram configuré et utilisable dans Home Assistant, il ne reste plus qu'à faire une automatisation !
La configuration yaml de cette automatisation se trouve dans le fichier joint "Notification Intrusion par Telegram".

![Telegram bot-BD](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/318790c0-2a7f-4a17-a330-c30a284d9363)

Ici une copie du message envoyé par HAOS par Telegram sur mon bot (il y a aussi une vidéo envoyée au bout de 45 secondes).

## 7. Ajout dans Alarmo de 2 actions liées aux alarmes (optionnel)
Ces 2 Automatisations créées à partir d'actions créées dans Alarmo servent à activer ou non les détections selon que l'alarme est déclenchée ou non.
Elles servent aussi à faire des essais de réglage en les déclenchant manuellement au lieu de mettre l'alarme (ici exemple où on démarre les détections et les enregistrements en appuyant sur ESSAI)
![démarrage détection](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/27ca58bb-26c4-4206-86a4-d4c22a7a08c8)

Les 2 actions sont configurées dans Alarmo et le fichier exemple est "Détection démarrage action Alarmoo
 - activation des détections caméras si Alarme armée (sert à faire des essais quand on veut)
  - désactivation des détections caméras si Alarme désarmée (sert à faire des essais quand on veut)

Si vous n'utilisez pas Alarmo vous devrez faire directement une Automatisation avec comme déclencheur la mise sous alarme de votre installation et comme action l'activation de switch.nom_de_votre_caméra_detect, qui est généré par Frigate.

# Ici des mises en place à moitié réussies
Le moitié réussi correspond à un problème de perte de droits au bout d'un certain temps - techniquement impossible d'utiliser le token de renouvellement des droits vers google-drive.
#### Si vous arrivez à faire fonctionner l'envoi des vidéos après l'expiration du token initial (donc avec les token de renouvellement) : MERCI DE M'INDIQUER CELA DANS LA PARTIE ISSUES DE CE GIT-HUB.

## 8. Téléchargement de l’add-on “rclone backup” 
Il va servir à exporter les photos et vidéos de personnes dans google-drive au fil de l'eau.
Ici encore il va falloir installer le lien dans les dépôts (Paramètres, Modules complémentaires, Boutiques des modules complémentaires, Trois petits  points en haut à droite, ajout de 
(https://github.com/jcwillox/hassio-rclone-backup) 
Puis redémarrage de HAOS, Boutique de modules complémentaires, Ajouter Rclone back-up.
Nota : on peut aussi l'intégrer directement à partir du bouton bleu dans le README du Github.

## 9. Configuration de l’add-on “rclone backup” 
Ma configuration de Rclone backup est particulière, j'ai dû "tordre" un peu l'addon car il fonctionne de base comme une sauvegarde à des moments fixés d'avance (jours, semaines, mois, années configurées dans un cron). 
Mon fonctionnement consiste à redémarrer Rclone back up à chaque détection et non une fois tous les jours.
La configuration est située dans la partie Configuration de l'addon/Module complémentaire Rclone back-up (onglets en haut) , ma configuration (accessible dans le fichier "Job pour Rclone"). Elle permet de lancer le backup à chaque démarrage de l'add-on (L'automatisation décrite dans le § suivant permettra de le relancer).
Description du job :
- pas de mention du moment de la sauvegarde (ligne effacée) pour permettre un backup à chaque démarrage de l'addon
- la commande est copyto qui copie à chaque lancement les fichiers nouveaux de Frigate vers Google drive (ce qui me permet de gèrer à la main les fichiers dans google drive - par exemple pour la destruction par exemple). Si vous souhaitez synchroniser automatiquement la source (HAOS) et la destination (Google Drive) qui auront toujours le même contenu, mettez à la place la commande rsynch.
- source : /media/frigate est l'endroit où Frigate met les fichiers par défaut.
- destination : google-drive:frigate. frigate est le répertoire à la racine de mon google drive où je fais mes sauvebardes.
-  
### ATTENTION : il faut configurer l'accès à Google drive par rclone ! 
Pour cela il faut obtenir de Google Drive pour Rclone un client id / client secret (qui sont créés sur son API de connection des applications) :la procédure complète se trouve sur https://rclone.org/drive/#making-your-own-client-id

Mais, sous HAos, je n'ai pas réussi à générer le token  qui et une troisième information nécessaire pour connecter une application à google drive. J'ai donc fait (comme Ryan72) une installation rclone sous Linux (en ligne de commande avec la commande rclone config) qui a fonctionnée tout de suite, ce qui m'a permis  afin de récupérer le token à partir des clientid/client secret déjà obtenus.
J'ai scrupuleusement copié les 3 informations (client id, client secret, token) ce qui m'a parmis de les integrer ensuite dans home assistant au niveau de la configuration d'accès de Rclone dans le fichier config/rclone.conf de votre HAOS.
Le fichier en exemple se nomme "Rclone.conf"
Si tout ça a bien marché vous pourrez voir que vous êtes connectés en cliquant sur Rclone back-up dans le panneau de gauche, en particulier en utilisant le menu de gauche Explorer et en "montant" Google Drive. Vous devez alors voir tous vos répertoires Gdrive dans le tableau. C'est la preuve que tout est bien configuré pour Rclone back-up.
Nota : après quelques jours, l'accès à Gdrive ne fonctionne plus ... en examinant le token d'accès j'ai vu qu'il avait seulement une journée de validité, mais qu'il y a un Refresh token qui prend je pense sa place ... INVESTIGATIONS EN COURS ... 
Bref, lorsque ça ne marche plus je reconfigure les accès à Gdrive (en gardant mes id/mot de passe, et le token change, c'est lui que je remets dans rclone.conf et c'est reparti !!!
### Re attention !
Après bien des essais, je n'ai pas réussi à faire en sorte que le Refresh Token soit efficace pour le rclone installé par rclone back-up. C'est pourqoi j'ai introduit dans l'automatisation Telegram l'envoi de la vidéo Frigate au bout de 45 secondes (pour permettre d'avoir une vidéo assez longue).

## 10. Construction d’une Automatisation pour relancer l’add-on "Rclone backup"
Ceci permet de relancer la copie vers Gdrive dès qu’un évènement (détection) est généré par Frigate, pour avoir des enregistrements sauvegardés dans Google drive assez vite après l’action (au cas où par exemple les intrus volent ou détruisent votre serveur HAOS).
L'automatisation est dans un fichier de ce tutoriel sous le nom "Redémarrage Rclone backup si détection humain".

Remarque :  j'ai mis 15 secondes de temporisation avant de démarrer la copie car les vidéos sauvegardées par Frigate dans HAOS sont découpées en tronçons de 10 secondes, comme ça la copie démarre avec au moins une vidéo à sauvegarder.
