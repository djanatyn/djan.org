+++
title = "creating a DIY privacy-respecting remote pet camera system"
description = "tailscale, frigate, openwrt"
date = 2026-11-14
draft = false

[taxonomies]
categories = ["lab notebook"]

[extra]
toc = false
code_copy = true
comment = false
+++

# the problem

i had a scary situation while traveling earlier this year where my cat went missing while i was on vacation! none of my petsitters could find domino, and he wasn't located until we got him and found him hiding in a closet in our building's basement.

as you can imagine, that was a traumatic experience for everybody involved. i was so happy to be reunited with domino, but so concerned by the lack of answers: when did he get out? how long had he been gone?

i'm happy to report that domino is in good health after that adventure, AND i created a small secure DIY pet camera system that i can access remotely when i travel!


# why not just use a normal pet camera?

i knew that i didn't want to use a commodity IoT pet camera device with a cloud-based service. these devices send audio and video of your home to servers outside of your control. i'm not comfortable with that level of trust, and [history shows](https://www.nytimes.com/wirecutter/reviews/wyze-security-breach/) that breaches can and do happen, with frightening outcomes:

{% <quote cite="[Cute Surveillance: Embodiment, Subjectivity, and Intimacy in Samanta Schweblin’s Little Eyes](https://www.researchgate.net/publication/397455313_Cute_Surveillance_Embodiment_Subjectivity_and_Intimacy_in_Samanta_Schweblin's_Little_Eyesem)"> %}
In February 2024, approximately thirteen thousand users of Wyze security cameras received footage of someone else’s home due to a malfunction in their device’s camera feed. Wyze’s second data breach in less than six months, the incident followed a glitch in September the previous year in response to which several consumers voiced their concern on the social news website Reddit. One person alarmingly wrote ‘I don’t know why, but I can see someone else’s camera,’ while another posted, ‘Apologies if this is your house.’
{% </quote> %}

outside of any unintentional breaches, pet cameras that send video and audio footage to a centralized cloud-based service will often retain your data, and may share that data with others without your consent:

{% <quote cite="[No One, Including Our Furry Friends, Will Be Safer in Ring's Surveillance Nightmare](https://www.eff.org/deeplinks/2026/02/no-one-including-our-furry-friends-will-be-safer-rings-surveillance-nightmare-0)">%}
The company has made law enforcement access a regular part of its business. As early as 2016, the company was courting police departments through free giveaways. The company provided law enforcement warrantless access to people’s footage, a practice they claimed to cut off in 2024. Not long after, though, the company established partnerships with major police companies Axon and Flock Safety to facilitate the integration of Ring cameras into police intelligence networks. The partnership allows law enforcement to again request Ring footage directly from users. This supplements the already wide-ranging apparatus of data and surveillance feeds now available to law enforcement.
{% </quote> %}

while that report was about Amazon's Ring cameras, and there are other cameras that exist which may respect your privacy more, **trusting an external service to handle your most intimate data** is the core problem.

even if you trust a service today, that service [could be sold to another company you *don't* trust](https://www.sec.gov/Archives/edgar/data/1018724/000101872419000004/amzn-20181231x10k.htm) next week!

this meant that i wanted to store **all of my video data locally**. none of the video goes to any external service: instead, i planned to [connect to my home network using tailscale](https://tailscale.com/) to access an [open-source self-hosted web interface](https://frigate.video/) to view the camera footage.

# don't trust the cameras

even if i wasn't using a pet camera product with a centralized service, i also wanted to avoid trusting the cameras themselves!

modern IP cameras are [IoT devices](https://en.wikipedia.org/wiki/Internet_of_things). they usually attempt to acquire IP addresses over DHCP, they host their own web interfaces, and they run their own proprietary firmware.

there is a rich tradition of exploiting security vulnerabilities on the firmware of IoT devices. the related works section from the paper ["A case of smart devices that compromise home cybersecurity"](https://www.sciencedirect.com/science/article/pii/S0167404824005923) 

to mitigate much of this risk, i chose to place my cameras on an isolated network segment, and to use my OpenWRT router's firewall to block communication with the manufacturer's servers.

# picking cameras

# buying a power-over-ethernet switch

# configuring firewall on `openwrt`

```
root@OpenWrt:~# cat /etc/config/firewall
config zone
	option name 'lan'
	list network 'lan'
	option input 'ACCEPT'
	option output 'ACCEPT'
	option forward 'ACCEPT'

config zone
	option name 'camera'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	list network 'camera'

config rule
	option src 'camera'
	option name 'Allow-DHCP-Camera-To-Router'
	list proto 'udp'
	option target 'ACCEPT'
	option src_port '68'
	option dest_port '67'
	option enabled '1'

config rule
	option src 'camera'
	option name 'Allow-DNS-Camera-To-Router'
	option dest_port '53'
	option target 'ACCEPT'
	option enabled '1'

config rule
	option src 'camera'
	option name 'Allow-NTP-Camera-To-Router'
	option dest_port '123'
	option target 'ACCEPT'
	option enabled '1'

config rule
	option src 'lan'
	option dest 'camera'
	option name 'Allow-Synology-To-Cameras'
	list dest_ip '192.168.40.0/24'
	option target 'ACCEPT'
	list src_ip '192.168.1.228'
	option enabled '1'

config rule
	option src 'lan'
	option dest 'camera'
	option name 'Allow-Mac-To-Camera'
	list src_ip '192.168.1.204'
	option target 'ACCEPT'
	option enabled '1'
```

# accessing the camera stream with `mpv`

```
mpv $"rtsp://synology:(pass show camera1/synology)@192.168.40.223:554/Streaming/Channels/101"
```

# setting the time with openwrt NTP

- show the lack of time as a sign that network connectivity was limited

# tweaking picture settings

- turning off audio recording
- suspicious settings for yelling at people

# setting up `frigate.nas.djan.org`

```
djanatyn@djan-nas:/etc/nginx$ sudo cat sites-enabled/nas-services.conf
Password:
server {
    listen 80;
    listen [::]:80;

    server_name
		nas.djan.org
        frigate.nas.djan.org
        paperless.nas.djan.org
        miniflux.nas.djan.org
        homarr.nas.djan.org
        concourse.nas.djan.org
        atuin.nas.djan.org
        qbittorrent.nas.djan.org
        jackett.nas.djan.org;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name nas.djan.org;

    ssl_certificate /usr/local/etc/nginx/certs/nas.djan.org/fullchain.pem;
    ssl_certificate_key /usr/local/etc/nginx/certs/nas.djan.org/key.pem;

    location / {
        proxy_http_version 1.1;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        proxy_ssl_protocols TLSv1.2 TLSv1.3;
        proxy_ssl_verify off;
        proxy_ssl_server_name off;
        proxy_ssl_session_reuse off;

        proxy_pass https://127.0.0.1:5001;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name frigate.nas.djan.org;

    ssl_certificate /usr/local/etc/nginx/certs/nas.djan.org/fullchain.pem;
    ssl_certificate_key /usr/local/etc/nginx/certs/nas.djan.org/key.pem;

    location / {
        proxy_http_version 1.1;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        proxy_ssl_protocols TLSv1.2 TLSv1.3;
        proxy_ssl_verify off;
        proxy_ssl_server_name off;
        proxy_ssl_session_reuse off;

        proxy_pass https://127.0.0.1:8971;
    }
}
```

# setting up `frigate` in docker

```
djanatyn@djan-nas:/volume1/media/frigate$ ls
config  docker-compose.yml  frigate
djanatyn@djan-nas:/volume1/media/frigate$ cat docker-compose.yml
services:
  frigate:
    image: ghcr.io/blakeblackshear/frigate:stable
    container_name: frigate
    restart: unless-stopped
    devices:
      - /dev/dri:/dev/dri
    shm_size: "512mb"
    ports:
      - "8971:8971"
      - "8554:8554"
      - "8555:8555/tcp"
      - "8555:8555/udp"
    environment:
      - FRIGATE_CAMERA_01_PASSWORD=${FRIGATE_CAMERA_01_PASSWORD}
      - FRIGATE_CAMERA_02_PASSWORD=${FRIGATE_CAMERA_02_PASSWORD}
    volumes:
      - ./config:/config
      - ./frigate:/media/frigate
      - /etc/localtime:/etc/localtime:ro
    tmpfs:
      - /tmp/cache:size=1g
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
djanatyn@djan-nas:/volume1/media/frigate$ sudo ls config
Password:
backup_config.yaml  frigate.db-shm
backup.db	    frigate.db-wal
config.yml	    go2rtc_homekit.yml
frigate.db	    model_cache
djanatyn@djan-nas:/volume1/media/frigate$ sudo cat config/config.yml
mqtt:
  enabled: false

detectors:
  cpu:
    type: cpu

record:
  enabled: true
  continuous:
    days: 3

snapshots:
  enabled: false

ffmpeg:
  hwaccel_args: preset-vaapi
  input_args: preset-rtsp-restream

detect:
  enabled: false
  fps: 2
  width: 640
  height: 360

go2rtc:
  streams:
    camera_01_main:
      - rtsp://synology:{FRIGATE_CAMERA_01_PASSWORD}@192.168.40.223:554/Streaming/Channels/101
    camera_01_sub:
      - rtsp://synology:{FRIGATE_CAMERA_01_PASSWORD}@192.168.40.223:554/Streaming/Channels/102
    camera_02_main:
      - rtsp://synology:{FRIGATE_CAMERA_02_PASSWORD}@192.168.40.201:554/Streaming/Channels/101
    camera_02_sub:
      - rtsp://synology:{FRIGATE_CAMERA_02_PASSWORD}@192.168.40.201:554/Streaming/Channels/102

cameras:
  camera_01:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/camera_01_main
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/camera_01_sub
          roles:
            - detect
    detect:
      enabled: false

  camera_02:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/camera_02_main
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/camera_02_sub
          roles:
            - detect
    detect:
      enabled: false

version: 0.17-0
```

# upgrading memory

```
djanatyn@djan-nas:~$ free -h
              total        used        free      shared  buff/cache   available
Mem:          9.6Gi       3.0Gi       465Mi       175Mi       6.1Gi       6.1Gi
Swap:         7.7Gi       363Mi       7.4Gi
```
