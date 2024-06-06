<h1 align="center">
  Hello! 👋
</h1>

This repository is the home for my Media Workflow **documentation**, meant to describe the process by which media is requested and moved between a local and remote media server.

## Table of Contents
- [Table of Contents](#table-of-contents)
- [Acknowledgment](#acknowledgment)
- [About The Project](#about-the-project)
- [What Problem Does This Solve?](#what-problem-does-this-solve)
- [Notes/To Do](#notesto-do)
- [Outline of Workflow](#outline-of-workflow)
  - [Jellyseerr](#jellyseerr)
  - [Sonarr/Radarr](#sonarrradarr)
  - [Queue4Download](#queue4download)
  - [MQTT with Mosquitto](#mqtt-with-mosquitto)
  - [Q4D Install Notes](#q4d-install-notes)
  - [Jellyfin (Remote)](#jellyfin-remote)
  - [Recap 1](#recap-1)
  - [Webhook](#webhook)
  - [Jellyseerr Webhook Notifications](#jellyseerr-webhook-notifications)

## Acknowledgment

This project relies heavily on the scripts created by weaselBuddha for the [Queue4Download project](https://github.com/weaselBuddha/Queue4Download/). This is an awesome project, so make sure to check it out for more information!

It also makes use of the [webhook tool](https://github.com/adnanh/webhook) created by adnanh. Make sure to check out that project as well.

Shout out to [@Mafyuh](https://github.com/Mafyuh) as well, whose Jellyseerr webhook idea helped me figure out how to tackle my own problem.

## About The Project

The goal of this project is to document the separate components which make up the whole media workflow. Since each component solves a specific problem, I think it makes sense for each to be housed in their own repository with this documentation repository providing a holistic view of the whole process.

This project is part of the broader [Infrastructure Documentation](https://github.com/chase-slept/infra-doc) project--please check it out if you're interested.

## What Problem Does This Solve?

The main problem is that the local network exists on an internet connection with a fair download speed but pretty terrible upload speed, making it impossible for users to watch media over the internet. The remote server solves our upload speed issue and allows users to access media via the internet, and while our remote server has very high bandwidth and upload/download speeds, it has limited storage space. It also introduces other problems, like having to deal with two separate media libraries, transferring files to-and-fro, tying everything into existing services like Sonarr/Radarr/Jellyseerr, etc. The biggest problem we introduce, however, is that remote users won't be able to request **existing** media with Jellyseerr---it will simply tell them the media already exists. So if it's on the local NAS already, we'll need to find a way to handle the request and transfer from the NAS to the remote server. To recap: we want both local and remote media requests to download on the remote server to make use of its speed; remote users want their requests stored on the remote server so they can access it over the internet; we want to copy our downloaded media to the NAS so local users can watch it without having to access the remote server and use its bandwidth; and we want remote users to be able to transfer existing media on-demand. Lastly, to mitigate the storage limitation on the remote server we can simply delete media as needed---since we copy it to the NAS, local users will still be able to access media removed from the remote server.

This is kind of a 'best of both worlds' where locally, users have a large library that makes use of the NAS and its higher-capacity (currently about 30TB), very high bandwidth (from gigabit ethernet and high-speed wireless connections), and very fast downloads via the remote server. Remote users will be able to make requests that are filled very quickly and are able to make use of the existing large NAS library as well. The only real downsides are that we now have to pay for another server to provide access to remote users, we have to deal with the added complexity introduced by tossing another server into the mix, and we have to spend some time and energy automating some things to address our 'wants' above.

## Notes/To Do

- Create write-up or README for media workflow
- Create/Tie in other repos/projects:
  - LFTP script x2
  - MQTT script and documentation
  - Webhook documentation
  - ~~sync-script~~

## Outline of Workflow

![Diagram of media workflow](assets/MediaWorkflowV1.png)

This diagram is a little bit busy, so we'll walk through it from where the workflow actually begins---the Jellyseerr request.

### Jellyseerr

The Jellyseerr instance is hosted on my local network, in a Docker container on a Raspberry Pi. It has access to the local Jellyfin library on the NAS, so it can see what media already exists when making new requests. It doesn't know anything about the remote Jellyfin library, but this won't cause any issues for our remote users. Both remote and local users will make media requests from this Jellyseerr instance, so they need to have user accounts. Counterintuitively, remote users are configured as 'Local Users' since they are not using the local Jellyfin instance---'Local User' in this context means a user that is configured within the Jellyseerr instance.  Local users that have a Jellyfin account can be configured as 'Jellyfin Users' or 'Local Users'. There are advantages to either configuration, but 'Jellyfin Users' must have a user profile on the local Jellyfin server. Check the Jellyseerr documentation for more info. Jellyseerr also has access to our local Sonarr/Radarr instances. Lastly, I've configured both Discord and Webhook notifications, but I'll only talk about the Webhook notifications as part of this workflow. More on that later. That's all of the relevant configuration information for Jellyseerr, so lets move on.

When a user puts in a media request in the Jellyseerr UI, it kicks off the workflow. Firstly, users will not be able to put in a request for media that already exists on the NAS; we'll come back to this in a bit. If the request is for new media, it's sent to either Sonarr or Radarr. Let's look at the relevant configuration for those next.

### Sonarr/Radarr

Both of these instances should have similar configurations. We're specifically interested in the 'Download Clients' and 'Media Management' settings. Our download client should be set to the torrent client on the remote server. I've set the 'category' as "shows" in Sonarr and "movies" in Radarr, and I'll explain why later. On this same page, at the bottom, we need to enable "Remove Completed", which will allow the apps to remove the download from our torrent client once it's completed downloading (and seeding, if seed limits are set in the Indexers settings--I've used Prowlarr to handle indexers and seed settings). Lastly, back in the Download Clients page, configure "Remote Path Mappings" to point to our remote torrent client's download path and set a **local** path where downloaded files go (this should be different from the media folder path). Later on we'll use this path, so remember it.

In the 'Media Management' settings, I've enabled "Hardlinks instead of Copy" and at the bottom of the page set the "Root Folders" to the **local** path to a media folder where Jellyfin 'sees' the files (this should be different from the download path). This is required for Sonarr/Radarr to be able to automatically import the media files after they are downloaded on the remote and transferred back to local server. For my setup, the NAS is mounted on the Raspberry Pi as an NFS drive, and that path is set as a mount point as part of the configuration for the Sonarr/Radarr Docker containers. If you're using Docker containers, check out [Trash Guides](https://trash-guides.info) for more information about hardlinks and how to configure your Docker containers to make use of them. This is a complicated topic on its own and out of scope of this documentation.

These apps do their thing and send the request to the torrent client on the remote server. Once the download completes it kicks off our first bit of automation and processing.

### Queue4Download

There are a few solutions for transferring files from a remote server to local server, like Rclone, Syncthing, Resilio Sync, FTP, custom scripts, etc. I wanted something mostly-or-entirely automated and ended up stumbling upon a project called [Queue4Download](https://github.com/weaselBuddha/Queue4Download). It consists of a handful of scripts on both the remote and local servers and offers some useful advantages over other tools/applications. First, it doesn't poll for updates but instead sends a message when downloads complete, so transfers start immediately instead of on a timer or interval. Second, it uses LFTP to initiate transfers. LFTP allows for segmented downloads, so once this is configured it vastly increases the transfer speeds from our remote server.

Unfortunately, Q4D is a little "tricky" to install and configure. The GitHub project page links to [these install instructions](https://www.reddit.com/r/sbtech/comments/1ams0hn/q4d_updated/) which worked well enough for me. I'll cover some details below as the individual components are relevant to understanding the whole workflow, but I won't be covering the installation line-by-line. Refer to the project page and linked instructions if you have any specific questions.

As mentioned above, rather than polling for completed files, Q4D relies on a messaging service. Picking up from where we left off with Sonarr/Radarr, once the download completes on our remote torrent client, the messaging service sends a message to the local server, where it is picked up by the matching message client. This initiates a script that starts an LFTP job to download the completed file. This whole process only takes a few minutes, which is incredibly quick!

### MQTT with Mosquitto

Messaging is handled by the Mosquitto message broker which uses the MQTT protocol. Installing typically requires sudo privileges, but a static binary is also available in the Q4D repository. My remote server doesn't allow sudo, so I'll proceed along that route. Using the binary is simple enough: stick it in `~/bin` along with the mosquitto_sub and mosquitto_pub binaries. We'll also need a config file which can be stored wherever (I eventually moved my mosquitto.conf and pws files into the .Q4D folder that is installed with that project). Mine looks like this:

```
listener 39399 0.0.0.0
persistence_file mosquitto.db
log_dest file /home/slept/.Q4D/mosquitto.log
log_type error
connection_messages true
log_timestamp true
allow_anonymous false
password_file /home/slept/.Q4D/mosquitto_pws
```

Note that the port can be changed as I've done if the server has limited port ranges to choose from. Next, create a password file `mosquitto_passwd -c /home/slept/.Q4d;mosquitto_pws  myUser` making sure to replace "myUser" with whatever user we're using. Lastly, create a systemd service file in `~/.config/systemd/user` and name it mosquitto.service. Mine looks like this:

```
[Unit]
Description=Mosquitto MQTT Broker Daemon

[Service]
Type=simple
ExecStart=/home/slept/bin/mosquitto -c /home/slept/.Q4D/mosquitto.conf

[Install]
WantedBy=multi-user.target
```

Make sure to `systemctl --user enable mosquitto.service`, then `start` then check for errors with `status`. The `--user` flag is important here since we're running this service as our user, not the root user. With all of this done, our MQTT broker is installed on the server, and we can continue on with the linked [install instructions](https://www.reddit.com/r/sbtech/comments/1ams0hn/q4d_updated/) for Queue4Download.

### Q4D Install Notes

The rest of the install and configuration is pretty straight forward via the instructions, so continue to follow those and make changes as needed (such as to configure SSH key authentication for LFTP, etc.). For my purposes I wanted to turn off the torrent labeling function and that didn't seem to work correctly. Even if I toggled the feature off where indicated in the script, it would still change the label to "DONE", which caused issues during testing. I would initiate a test transfer, it would fail but perplexingly set the label as DONE, meaning I had to change the label back manually before I could test or transfer it again. Since the initial label is what the logic for transferring files is based on, I decided to disable this feature by commenting out a line of code in the Q4Ddefines.sh file: `#readonly LABEL_CHANNEL="Label"`. I didn't really need the feature anyway, so disabling this was just a personal preference to make testing easier.

I mentioned earlier that I set categories in Sonarr and Radarr. We likewise edit the Types.config file on the server-side to refer to these labels and assign them a shorthand type-code. On the client side, the Q4Dclient.sh file translates these type-codes to the relevant **local** file paths. These paths are where our media files eventually get transferred once the LFTP job finishes and we need them to match the Sonarr/Radarr **download** paths that we set earlier (that is to say, the paths where the apps expect newly downloaded files to exist locally). What this ends up looking like for my setup, with the type-code path matching the local download folders:

App | Remote DL | Local DL | Media Folder
--- | --- | --- | ---
Sonarr | /home/slept/downloads/rtorrent | /data/torrents/tv | /data/media/tv
Radarr | /home/slept/downloads/rtorrent | /data/torrents/movies | /data/media/movies

Once everything is configured, be sure to test that everything is working by following the 'test' section in the instructions! The one-liner mentioned there `bash -xv ~/.Q4D/Queue4Download.sh Param1 Param2 Param3...` can also be used to manually initiate the transfer process if for some reason it has failed for a specific torrent (`Param1` in my case was all that is needed, with the parameter being the torrent name--easily copied from the right-click menu in ruTorrent). I found it helpful to `tail -f` the mosquitto.log and queue.log files on the remote server and the Events.log and process.log files on the local server while testing, to view the messages and make sure they were translating to actions in the scripts. If things aren't working as expected, thoroughly read the comments on the linked instructions or message the script author---he seems to be actively maintaining the project and can probably help.

### Jellyfin (Remote)

Since our remote users are using the high-bandwidth server for Jellyfin, we need to configure a few things. In the ruTorrent settings under Autotools, I enabled the "AutoMove with a filter to match `/movies|shows/` and set the path to finished downloads to `/home/slept/media`, change the operation type to 'hard link', and enable "Add torrent's label to path". This is the path we'll use for media files in the Jellyfin instance that runs on the remote server. Hardlinking the completed files to this directory will prevent us from having to actually move or copy any files, which will allow the torrents to continue seeding if needed and prevent duplicate files from taking up our limited storage. When it comes time to delete files to make room, we'll only need to remove files from this media folder as, presumably, Sonarr/Radarr will have removed the original download files since we configured it to do so earlier. Create user accounts for any remote users and that completes the configuration for Jellyfin. Remember, our local Jellyseerr instance doesn't interface with this Jellyfin at all, so the user accounts we create won't be able to be used with Jellyseerr--a separate user account will need to be made for remote users to create media requests.

### Recap 1

So to recap where we are now in relation to our diagram. Remote or local users create a media request in Jellyseerr. This gets sent to Sonarr/Radarr, which processes the request and sends the download to the remote download client with a "shows" or "movies" label. Once the download completes, the torrent client sends a trigger to our MQTT broker, which uses the label to assign a type-code along with the torrent hash to create a message. This message is queued and published by the broker on the remote server and the subscribed client (our local server) receives the message and runs the Q4D client script. The script spawns the LFTP job to download the file(s) from the remote server to the relevant path on the local server. This whole process should take just a few minutes. So far, we've accomplished most of what we wanted in our big list of problems to solve above. All that's left is to deal with transferring **existing media** from the local server to the remote, which is sort of the inverse of what we just accomplished. A big caveat here as it pertains to my local network: since my upload speed is so low, segmenting the downloads with LFTP actually isn't very helpful, and just tanks my home network speeds. I've addressed this by warning my users and adding a message when they make requests. One day I'll have better upload speed, and this whole workflow will become essentially moot, haha! Until then, lets continue with the elaborate workaround.

### Webhook

The main consideration with moving existing files was how to initiate the request. Jellyseerr handles new media perfectly and it's an entirely automated pipeline. I wanted something similar to work for media that already exists on my local NAS. I ended up stumbling across [someone else's solution](https://old.reddit.com/r/selfhosted/comments/17sojv5/automate_jellyfinjellyseer_issues/) for automating actions in Jellyfin/Jellyseerr by using the 'Issues' and Webhooks features in Jellyseerr. This is pretty much exactly what I needed to tackle this problem.

Firstly, Jellyseerr will indicate what show/movies are already available. We've tied Jellyseerr to the **local** Jellyfin, which looks at our local NAS, available only to local users. If a remote user wants some existing media, Jellyseerr simply says its already available, but obviously it doesn't exist on the remote server yet. Instead, we have the remote user submit an Issue in Jellyseerr. On the media they want to request, there is a 'Report issue' button they can use to do this easily. I've instructed users to add "unavailable" as the issue message, for use in the script I've created (more on this later).

On my local Raspberry Pi, I installed [webhook](https://github.com/adnanh/webhook). The version available on the DietPi default repositories is wildly out of date, so I manually installed the latest binary from the repository, then created a systemd service file to run it on startup. Note that we need to create the JSON/YAML file that `-hooks` below references before we start this service, which we'll do next. This next bit of instruction is confusingly out-of-order as most of the script and hooks file came together while testing and tinkering. Bear with me.

```
[Unit]
Description=Webhook Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/webhook -hooks /etc/webhook.conf -port 9990 -verbose
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

For the `webhook.conf` file, which is actually a YAML file (note that I've left out the trigger-rule, check the documentation for more info):

```yaml
- id: script-webhook
  execute-command: /home/slept/scripts/webhooks/test.sh
  command-working-directory: /home/slept/scripts/webhooks
  pass-arguments-to-command:
    - source: payload
      name: media.media_type
    - source: payload
      name: subject
#    - source: payload
#      name: media.tmdbId
#    - source: payload
#      name: media.tvdbId
    - source: payload
      name: issue.issue_id
    - source: payload
      name: message
```

The YAML file references some payload information that we'll pull from Jellyseerr in just a bit. We'll also need to create the script that's referenced in `execute-command:`. This is what triggers when a webhook is received. My final working script is available [here](https://github.com/chase-slept/sync-script/blob/main/syncV3.sh). I won't paste the whole script here but I'll post some excerpts to help explain as needed. Once everything is created we can start the service with `systemctl` and continue along.

### Jellyseerr Webhook Notifications

In Jellyseerr's settings, under "Notifications", I've enabled the Webhook agent and selected the "Issue Reported" and "Issue Reopened" notification types. For the Webhook URL, we can use the local IP of our Pi since Jellyseerr and the Webhook server are on the same network (or a FQDN if you have one--I used Caddy to reverse proxy to one) along with `/hooks/{id}` replacing `id` with the value we named in the hooks file above. The full URL should be `http://10.10.10.10/hooks/script-webhook`. Test that it's working and save; no other changes need to be made here.


More documentation pending ...