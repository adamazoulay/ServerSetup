1. Proxmox
  I'm using [Proxmox](https://www.proxmox.com/en/) as my virutualization environment on my baremetal server since it's FOSS, same feature set as ESXi, and has ZFS support so I can easily share my pool among VMs. Set this up on the server following a guide online.


2. ZFS Pool
  Once Proxmox is set up on the bare metal, we boot it up and set up the zfs pool. Again, lots of guides online to do this. The important part is making sure you use the correct drives and have RAID configured how you want (mirrored is ideal). Once we have the pool set up, we must enable the NFS sharing. This is easy with zfs since it's built in:

  !!NOTE!!: This leaves the share wide open. Easy for permissions but bad for security. Make sure there is no out-of-network access or else you could lose everything.
  ```
  # On host (Proxmox in our case):
  apt-get install -y nfs-kernel-server  # Install nfs server if needed
  zfs set sharenfs=rw,no_root_squash tank  # Assuming the pool is called 'tank'
  showmount -e  # To verify the share is working

  # On client (replace $host_ip with host's ip):
  sudo apt-get install nfs-common
  sudo mount -t nfs $host_ip:/tank /tank
  ```

Helpful zpool commands:
  ```
  zpool export tank  # Remove pool from system (for moving pool)
  zpool import tank  # Add pool to system
  zpool status  # Tank info (also show on proxmox dash)
  ```

  Once you have everything set up you can add the pool into the proxmox webUI by adding storage->ZFS and selecting your pool.


3. Install Ubuntu VM
  Okay, this bad boy is the workhorse of the server. It will host all the Docker images for you including plex and the nginx reverse proxy. Make sure it has at least 8GB of ram and a decent number of CPUs for transcoding on plex. Since we are running ZFS on Proxmox we can divide up our RAM nicely between the two (since more RAM on ZFS host -> more speed serving from RAM)
  
  Don't forget to mount the ZFS pool once the VM is up! Set to mount at load like this:
  ```
  sudo nano /etc/fstab
  $host_ip:/tank       /tank      nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0  # Place this line in fstab
  ```

4. Docker install:
  ```
  sudo apt-get update
  sudo apt-get upgrade
  sudo apt install docker.io  # Install docker
  sudo systemctl start docker  # Start docker
  sudo systemctl enable docker  # Start at system startup
  ```


5. Set system variable to point to program dir (mine is at /home/programs/)
  ```
  sudo touch /etc/profile.d/docker_data.sh
  sudo nano /etc/profile.d/docker_data.sh
  ```
  Inside nano, type ```export docker_data=/home/programs```, then ctrl-s and ctrl-x to save and exit.


6. Portainer:
  ```
  sudo docker run -d -p 9000:9000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v $docker_data/portainer/data:/data portainer/portainer
  ```
  Go to the sever-ip:9000 to access portainer web-ui. Create a user, and on the next screen select 'Local' and hit connect. Click 'Endpoints' on the left and click 'local'. Then enter the server ip address into the 'Public IP' field. Click update.


7. Nginx
  ```
  sudo docker run --name nginx --network host --restart always -v $docker_data/nginx/letsencrypt:/etc/letsencrypt -v $docker_data/nginx/conf.d:/etc/nginx/conf.d -d nginx
  ```
  Copy .conf files for all sites.

  Install certbot inside the nginx docker:
  ```
  sudo docker exec -it nginx bash  # This will work any time you want to enter a containers shell
  apt-get update
  apt-get install software-properties-common gpg -y
  add-apt-repository "deb http://archive.ubuntu.com/ubuntu $(lsb_release -sc) universe"
  add-apt-repository ppa:certbot/certbot
  apt-get update
  apt-get install certbot python-certbot-nginx -y
  ```
  
  You can run the certification with ```certbot --nginx```


6. Plex

  !!!!!Make sure PUID and PGID match your user IDs!!!!!
  ```
  sudo docker run -d \
  --name=plex \
  --net=host \
  --restart=always \
  -e VERSION=latest \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -v $docker_data/plex/config:/config \
  -v /tank/media/tvshows:/data/tvshows \
  -v /tank/media/tvshowsanime:/data/tvshowsanime \
  -v /tank/media/movies:/data/movies \
  -v $docker_data/plex/transcode:/transcode \
  linuxserver/plex:public
  ```
  
  Note: Metadata won't work unless you uncheck 'local media' under agents in the plex server settings.


7. QBittorrent (Built in VPN)
  I use surfshark.
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
  You need to put the .opvn config file in the openvpn folder under config, and restart the container.

8. Jackett
```
sudo docker run -d \
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

9. Sonarr
```
sudo docker run -d \
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

10. Sonarr - Anime
```
sudo docker run -d \
  --name=sonarr-anime \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -e UMASK_SET=022 `#optional` \
  -p 8988:8989 \
  -v $docker_data/sonarr-anime:/config \
  -v /tank/media/tvshowsanime:/tvshows \
  -v /tank/downloads:/downloads \
  --restart always \
  linuxserver/sonarr
```


11. Radarr
```
sudo docker run -d \
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


12. Ombi
```
sudo docker run -d \
  --name=ombi \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Toronto \
  -p 3579:3579 \
  -v $docker_data/ombi/config:/config \
  --restart always \
  linuxserver/ombi
```


13. OpenVPN
```
Use the openvpnas esxi image
```


14. Nextcloud

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
Inside cloud.galactica.conf in nginx, add ```client_max_body_size 100M;``` inside server block.

Import copied files with: ```sudo docker exec nextcloud sudo -u abc php /config/www/nextcloud/occ files:scan --all```



15. Jellyfin (with Lazyman)

Can use 'rebecca' as un, no password to log in with current config.

Use 'http://freegamez.ga/' to get ip for hosts (at time of writing: 165.22.201.101)
```
sudo docker run -d \
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
sudo docker run -d \
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

17. Factorio
```
sudo docker run -d \
  -p 34197:34197/udp \
  -p 27015:27015/tcp \
  -v $docker_data/factorio:/factorio \
  --name factorio \
  --restart=always \
  factoriotools/factorio:stable
```

18. Minecraft
```
sudo docker run -d \
  -e EULA=TRUE \
  -p 25565:25565 \
  -v $docker_data/minecraft:/data \
  --name mc \
  -e DIFFICULTY=normal \
  -e 'MOTD=Get on the crafting and the mining' \
  -e PVP=false \
  -e ONLINE_MODE=FALSE \
  itzg/minecraft-server
```



!!OLD!!

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
