0. ZFS Pool
Set up a zfs pool (lot's of guides online to do this)

To remove from system:
```
zpool export tank
```

To add to system:
```
zpool import tank
```

1. Docker install:
```
sudo apt-get update
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

2. Set system variable to point to program dir (mine is at /home/programs/)
```
sudo touch /etc/profile.d/docker_data.sh
sudo nano /etc/profile.d/docker_data.sh
```
  Inside nano, type ```export docker_data=/home/programs```, then ctrl-s and ctrl-x to save and exit.


3. Portainer:
```
sudo docker run -d -p 9000:9000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v $docker_data/portainer/data:/data portainer/portainer
```
  Go to the sever-ip:9000 to access portainer web-ui. Create a user, and on the next screen select 'Local' and hit connect. Click 'Endpoints' on the left and click 'local'. Then enter the server ip address into the 'Public IP' field. Click update.

4. Wordpress

Make uploads.ini in programs dir
```
file_uploads = On
memory_limit = 64M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
```

Stacks in portainer, add stack:
```
version: '2'
services:
  wordpress:
    image: wordpress:latest # https://hub.docker.com/_/wordpress/
    ports:
      - 30001:80 # change port for every deployment. Get nginx config to look here
    volumes:
      - /home/programs/rebeccabruzzesecom/wordpress/uploads.ini:/usr/local/etc/php/conf.d/uploads.ini
      - /home/programs/rebeccabruzzesecom/wordpress/wp-app:/var/www/html # Full wordpress project
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: root
      WORDPRESS_DB_PASSWORD: paaASD34DSfhsop347yasSDDertg # Change this
    depends_on:
      - db
    networks:
      - internal
  db:
    image: mysql:5.7
    ports:
      - 3315:3306 # change port for every new deployment
    volumes:
      - /home/programs/rebeccabruzzesecom/mysql:/var/lib/mysql
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_ROOT_PASSWORD: paaASD34DSfhsop347yasSDDertg # Change this
    networks:
      - internal
networks:
  internal:
    driver: bridge
```

  Open wordpress and install with any setting. We will restore backup later.

5. Nginx
```
sudo docker run --name nginx --network host --restart always -v $docker_data/nginx/letsencrypt:/etc/letsencrypt -v $docker_data/nginx/conf.d:/etc/nginx/conf.d -d nginx
```
  Copy .conf files for all sites.

  Install certbot inside the nginx docker:
  ```
  apt-get update
  apt-get install software-properties-common gpg -y
  add-apt-repository "deb http://archive.ubuntu.com/ubuntu $(lsb_release -sc) universe"
  add-apt-repository ppa:certbot/certbot
  apt-get update
  apt-get install certbot python-certbot-nginx -y
  ```

6. Plex

!!!!!Make sure PUID and PGID match your user IDs!!!!!
```
sudo docker create \
--name=plex \
--net=host \
--restart=always \
-e VERSION=latest \
-e PUID=1000 \
-e PGID=1000 \
-e TZ=America/Toronto \
-v $docker_data/plex/config:/config \
-v /tank/media/tvshows:/data/tvshows \
-v /tank/media/movies:/data/movies \
-v /tank/media/transcode:/transcode \
linuxserver/plex
```

7. QBittorrent (Built in VPN)
```
sudo docker run --privileged  -d \
              --name=qbittorrentvpn \
              -v $docker_data/qbittorrent/config:/config \
              -v /tank/downloads:/downloads \
              -e "VPN_ENABLED=yes" \
              -e "LAN_NETWORK=192.168.0.0/24" \
              -e "NAME_SERVERS=8.8.8.8,8.8.4.4" \
              -e "VPN_USERNAME=un" \
              -e "VPN_PASSWORD=pw" \
              -e "PUID=1000" \
              -e "PGID=1000" \
              -p 8080:8080 \
              -p 8999:8999 \
              -p 8999:8999/udp \
              --restart always \
              markusmcnugen/qbittorrentvpn
```

8. Sonarr
```
sudo docker create \
  --name=sonarr \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -e UMASK_SET=022 `#optional` \
  -p 8989:8989 \
  -v $docker_data/sonarr:/config \
  -v /tank/media/tvshows:/tvshows \
  -v /tank/downloads:/downloads \
  --restart always \
  linuxserver/sonarr
```

9. Jackett
```
sudo docker create \
  --name=jackett \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -p 9117:9117 \
  -v $docker_data/jackett:/config \
  -v /tank/downloads:/downloads \
  --restart always \
  linuxserver/jackett
```

10. Radarr
```
sudo docker create \
  --name=radarr \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -p 7878:7878 \
  -v $docker_data/radarr:/config \
  -v /tank/media/movies:/movies \
  -v /tank/downloads:/downloads \
  --restart always \
  linuxserver/radarr
```

11. Ombi
```
sudo docker create \
  --name=ombi \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -p 3579:3579 \
  -v $docker_data/ombi/config:/config \
  --restart always \
  linuxserver/ombi
```

12. OpenVPN
```
Use the openvpnas esxi image
```

13. Nextcloud

Add stack:
```
version: "2"
services:
  nextcloud:
    image: linuxserver/nextcloud
    container_name: nextcloud
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /home/programs/nextcloud/config:/config
      - /tank/cloud:/data
    ports:
      - 1001:443
    restart: always
    networks:
      - internal   
  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: 'db'
      MYSQL_USER: 'nextcloud1'
      MYSQL_PASSWORD: 'nextcloud1'
      MYSQL_ROOT_PASSWORD: 'nextcloud1'
    ports:
      - '3316:3306'
    volumes:
      - /home/programs/nextcloud/mysql:/var/lib/mysql
    networks:
      - internal
networks:
  internal:
    driver: bridge
```
Need to remove ssl internally, so go into $docker_data/nextcloud/config/nginx/site-config and edit default. Change 'listen 443 ssl;' to 'listen 443;'

Also add to config.php:
```
'overwrite.cli.url' => 'https://cloud.galactica.host',
'overwriteprotocol' => 'https',
'overwritehost' => 'cloud.galactica.host',
```
Import copied files with: ```sudo docker exec nextcloud sudo -u abc php /config/www/nextcloud/occ files:scan --all```

Inside cloud.galactica.conf in nginx, add ```client_max_body_size 100M;``` inside server block.

14. Emby (with Lazyman)

Use 'http://freegamez.ga/' to get ip for hosts (at time of writing: 178.62.203.238)
```
sudo docker create \
  --name=emby \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -e UMASK_SET=000 `#optional` \
  -p 8096:8096 \
  -v $docker_data/emby:/config \
  --add-host mf.svc.nhl.com:178.62.203.238 \
  --add-host mlb-ws-mf.media.mlb.com:178.62.203.238 \
  --add-host playback.svcs.mlb.com:178.62.203.238 \
  --restart always \
  linuxserver/emby
```

15. Jellyfin (with Lazyman)

Use 'http://freegamez.ga/' to get ip for hosts (at time of writing: 165.22.201.101)
```
sudo docker create \
  --name=jellyfin \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -e UMASK_SET=022 `#optional` \
  -p 8096:8096 \
  -p 8920:8920 `#optional` \
  -v $docker_data/jellyfin:/config \
  -v /tank/media/tvshows:/data/tvshows \
  -v /tank/media/movies:/data/movies \
  -v /tank/media/transcode:/transcode `#optional` \
  --add-host mf.svc.nhl.com:165.22.201.101 \
  --add-host mlb-ws-mf.media.mlb.com:165.22.201.101 \
  --add-host playback.svcs.mlb.com:165.22.201.101 \
  --restart always \
  linuxserver/jellyfin
```

16. Tatulli

```
sudo docker create \
  --name=tautulli \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -p 8181:8181 \
  -v $docker_data/tatulli/config:/config \
  -v $docker_data/tatulli/logs:/logs \
  --restart unless-stopped \
  linuxserver/tautulli
```

17. TinyMediaManager

Access webUI at 5800
```
sudo docker run -d --name=tinymediamanager \
-v $docker_data/tinymediamanager/config:/config \
-v /tank/downloads:/media:rw \
-e GROUP_ID=1000 -e USER_ID=1000 -e TZ=America/Toronto \
-p 5800:5800 \
romancin/tinymediamanager:latest
```

18. Bionic client

```
sudo docker run -d \
  --name boinc \
  --net=host \
  --pid=host \
  -v /tank/boinc:/var/lib/boinc \
  -e BOINC_GUI_RPC_PASSWORD="123" \
  -e BOINC_CMD_LINE_OPTIONS="--allow_remote_gui_rpc" \
  boinc/client
```
