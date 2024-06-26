# HAOS-Frigate-addon-Install-Notif-Backup
## WARNING
### [For the French version please click Here go to README.md](README.md)

## Introduction
There is extensive documentation available for installing Frigate in an LSX Container, but I couldn't find one for installing a Frigate add-on suite with Backup and smartphone notifications.

Having struggled a bit and exchanged on these subjects, I think it's interesting to share what works for me with others.

## Prerequisites
### It is assumed that you have already:
- Installed Proxmox
- Installed HAOS
- Installed the following add-ons: HACS, Mosquitto browser, Alarmo (optional)
- Set up your external access (optional): for me, with DuckDNS and NGINX add-ons
- Tested your cameras with the rtsp or onvif protocol (or another, but I haven't tested)
- Configured a Telegram bot in HAOS (optional).

### Tutorial outline:
- Connecting Coral in the HAOS VM
- Installing Frigate add-on
- Frigate Configuration
- Integration of Frigate (yes, it's different, I had trouble understanding this)
- Creating an Automation to send a notification with text, photos, and videos via Telegram_bot
- Configuring notifications using SgtBatten's blueprint
- Adding in Alarmo (or directly constructing by Automations) 2 actions related to alarms:
  - activating camera detections if the Alarm is armed (also used for testing)
  - deactivating camera detections if the Alarm is disarmed (also used for testing)
#### Also, a partially successful attempt at backing up to Google Drive:
The partially successful part corresponds to a problem of rights loss after a certain time - technically impossible to use the token for renewing rights to Google Drive:
- Downloading the "rclone backup" add-on, which will be used to export photos and videos of people to Google Drive
- Configuration of the "rclone backup" addon
- Building an Automation to restart the "Rclone backup" add-on whenever an event (detection) is generated by Frigate, to have recordings backed up to Google Drive quite quickly (15 seconds) after the intrusion.

## 1. Connecting Coral in the HAOS VM
### Initial Proposal
There are multiple tutorials on "How to access Coral from Frigate," all based on integration from the USB port number.
Here I show a method that is usually not used, because I preferred to do an integration based on the name or ID of the Coral component, especially because I am not a Linux pro so I had difficulties identifying the USB port, integrating it through the Proxmox interface, etc.
I have only put one photo because I think you must have already integrated USB dongles into your Home Assistant OS.

So, after connecting your CORAL Key - with a USB cable and not directly to your computer (for me a NUC), you need to:
- go to the Proxmox interface,
- click on your HAOS VM
- go to the "Hardware" line
- click on "Add" at the top, then "Add USB device"
- choose from the drop-down menu "Use USB device and manufacturer IDs"
- finally choose the device corresponding to Coral.
### ATTENTION: you will need to do this manipulation twice because the Coral device changes its name.
- the first time for me it was seen as GOOGLE manufacturer with an id
- after some Frigate tests it had disappeared from the config and so it had to be reinstalled, this time it didn't have a manufacturer name, you have to identify it with its id (which also changed)
- after this second integration it is stable for me and recognized in Frigate (we will see later how to be sure).

### More conventional proposal (found in various tutorials)
This involves integrating Coral from the USB port number.... manipulation successful for me by "breaking" the method above.
First, you need to identify the various USB dongles present on your "HA server" (for me a NUC).
For this in Proxmox:
- go back to the node level where your HAOS VM is located,
- select the shell of this node (for me I called it Proxmox)
- run the lsusb command, which will give you the different ports used

![Coral 1 lsusb](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/f2a430a9-2583-43cb-a7e8-454c44d914fb)
- then go to the interface of the HAOS VM (for me VM 102 - VM Prod), and integrate Coral as above but this time choosing "use USB port", which gives in detail:
  - go to the Proxmox interface,
  - click on your HAOS VM
  - go to the "Hardware" line
  - click on "Add" at the top, then "Add USB device"
  - choose from the drop-down menu "Use the USB port"
  - finally choose the device corresponding to Coral, by clicking on the correct line 'for me I chose line 2.6 while lsusb gave me usb 2 port 5 !!! I deduced it by identifying all the other dongles/USB ports)
  ![Coral 2 vision VM HAOS](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/b69e1a73-04cc-4dae-a8ef-49241d6e687e)

 
### !!! In both installation cases you can move on to installing the Frigate add-on, in the next section.

## 2. Installing Frigate add-on

- In your Home Assistant, go to Settings, Add-ons, Add-on Store, then click at the top right to add a new Add-ons repository and add this link https://github.com/blakeblackshear/frigate to the list.
- Restart Home Assistant
- Go to Settings, Add-ons, Add-on Store (at the bottom for me), search for Frigate and click on Frigate, follow the instructions indicating that you are running it on HAOS.

- Go back to Settings, Add-ons, and check "Show in sidebar"
  
 ![Frigate Add-on](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/1ea7e5a3-8082-4348-9e0e-3d1826be2c57)

From this moment on, you can configure Frigate.

## 3. Building the Frigate Configuration
Since you have Frigate in the sidebar (on the left for me), that's where we'll configure it.
For this, I'm attaching my configuration file, it's called "Config Frigate addon oldchap56" and the comments are inside. For details on the content, I recommend the excellent tutorial from Raynox, which it's based on, which you can find at https://www.youtube.com/watch?v=-haxDKIOEao

In summary, the chosen options allow each detection of a human by Frigate (done on the low-resolution stream) to take a snapshot and a video with the high-resolution stream. 
These snapshots and videos are accessible:
- either in the Frigate interface, by clicking on Events, we have all the detections classified chronologically,
- or in the Media interface, classified by date then camera then again chronology.

Likewise, to check that Coral has been recognized by Frigate, you can:
- click on Frigate in the left sidebar,
- then go to the "System" tab,
- and there you can verify that Coral is working fine !!

![Frigate Interface to see Coral](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/3f1e4fcc-942f-4fc0-8709-04150a5ef993)

  
## 4. Integration of Frigate
This integration allows you to use Frigate's sensors, events, etc. which communicate via mqtt with the rest of the world (HAOS, cameras, etc.). This is what allows you to create Automations based on what happens inside Frigate.
For integration, go to HACS (left sidebar), Integrations, click on Custom Repositories to include the following link in the Integration category
https://github.com/blakeblackshear/frigate-hass-integration.
Then, click on it to install. [TO BE VALIDATED]
Note: you can also integrate it directly from the blue button in the Github README.

## 5. Configuring notifications using SgtBatten's blueprint
Nothing complicated for this, it allows you to receive a notification on your smartphone in case of intrusion with a snapshot and by clicking on it access to the videos through the mobile application of Home Assistant.
To do this:
- go to Settings, Automations, Blueprint and search for SgtBatten's Frigate Notifications automation.
- create an Automation using the Blueprint. I put my Yaml file for this Automation in the attached file "Intrusion Detection & HA Notification"

![Notification HA-BD](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/51406148-95b9-4608-9c1f-4dfac72a58a3)

Here the notification by HAOS on my phone.

But, this was not enough for me because apparently you can only send the Notification to 1 phone (I failed to make a Group of phones in HA).
So, I made another Automation that sends the information to a Telegram bot (SEE next §). Thus all family members who are part of it will receive the alert in case of intrusion.


## 6. Creating an Automation to send a notification with text, photos, and videos via Telegram_bot (optional)
To do this you need to have configured a Telegram Bot. If not done, I recommend this Youtube tutorial by Maternix (https://www.youtube.com/watch?v=gJpnIslsLqU)

Now that you have your Telegram bot configured and usable in Home Assistant, all that remains is to create an automation!
The yaml configuration of this automation is in the attached file "Intrusion Notification by Telegram".

![Telegram bot-BD](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/318790c0-2a7f-4a17-a330-c30a284d9363)

Here a copy of the message sent by HAOS via Telegram to my bot (there is also a video sent after 45 seconds).

## 7. Adding in Alarmo 2 actions related to alarms (optional)
These 2 Automations created from actions created in Alarmo are used to activate or not the detections depending on whether the alarm is triggered or not.
They also serve to adjust the settings by triggering them manually instead of setting the alarm (here example where we start the detections and recordings by pressing TRY)
![start detection](https://github.com/oldchap56/HAOS-FrigateAddon-Coral-Install-Notif-Backup/assets/153823477/27ca58bb-26c4-4206-86a4-d4c22a7a08c8)

The 2 actions are configured in Alarmo and the example file is "Detection start action Alarmoo"
 - activating camera detections if Alarm is armed (used for testing)
  - deactivating camera detections if Alarm is disarmed (used for testing)

If you do not use Alarmo, you will have to directly create an Automation with the trigger being the arming of your installation's alarm and the action being the activation of switch.your_camera_detect, which is generated by Frigate.

# Here are partially successful setups
The partially successful part corresponds to a problem of rights loss after a certain time - technically impossible to use the token for renewing rights to Google Drive.
#### If you manage to make the sending of videos work after the expiration of the initial token (thus with the renewal tokens): THANK YOU TO LET ME KNOW IN THE ISSUES OF THIS GIT-HUB.

## 8. Downloading the “rclone backup” add-on 
It will be used to export photos and videos of people to Google Drive in real-time.
Here again you will need to install the link in the repositories (Settings, Add-ons, Add-on Store, Three small dots in the top right corner, adding 
(https://github.com/jcwillox/hassio-rclone-backup) 
Then restart HAOS, Add-on Store, Add Rclone back-up.
Note: you can also integrate it directly from the blue button in the Github README.

## 9. Configuration of the “rclone backup” add-on 
My Rclone backup configuration is special, I had to "bend" the addon a bit because it works by default as a backup at fixed times (days, weeks, months, years configured in a cron). 
My operation consists of restarting Rclone back up with each detection and not once a day.
The configuration is located in the Configuration part of the addon / Rclone back-up (top tabs), my configuration (accessible in the file "Job for Rclone"). It allows to launch the backup at each launch of the addon (The automation described in the next § will restart it).
Job description:
- no mention of the time of the backup (line deleted) to allow a backup with each launch of the addon
- the command is copyto which copies new files from Frigate to Google Drive each time it starts (which allows me to manually manage the files in google drive - for example for destruction). If you want to automatically synchronize the source (HAOS) and the destination (Google Drive) which will always have the same content, put the rsynch command instead.
- source: /media/frigate is where Frigate puts the files by default.
- destination: google-drive:frigate. Frigate is the directory at the root of my google drive where I do my backups.

## 10. Building an Automation to restart the “Rclone backup” add-on 
This will be restarted whenever an event (detection) is generated by Frigate, to have recordings backed up to Google Drive quite quickly (15 seconds) after the intrusion.
Here's the yaml configuration of this automation (accessible in the "Intrusion Detection & HA Notification" file): a simple call to the service restart of the addon (addon_restart).
