+++
title = "creating a DIY privacy-respecting remote pet camera system"
description = "tailscale, frigate, openwrt"
date = 2026-11-14
draft = true

[taxonomies]
categories = ["lab notebook"]

[extra]
toc = false
code_copy = true
comment = false
<!-- og_image = "/img/zig_inkplate_hello_world.png"  -->
+++

# the problem

# don't let the camera do evil stuff

# picking cameras

# buying a power-over-ethernet switch

# configuring firewall on `openwrt`

- creating zones
- assigning a static IP to the cameras
- assigning a static IP to 

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
