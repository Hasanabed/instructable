Raspberry Pi Install
====================

## Kernel parameters

Add to `/boot/cmdline.txt`

    cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory swapaccount=1 elevator=deadline rootflags=data=writeback,commit=120 quiet

Reboot

    pi@raspberrypi:~ $ cat /proc/cgroups
    #subsys_name    hierarchy       num_cgroups     enabled
    cpuset  5       16      1
    cpu     2       121     1
    cpuacct 2       121     1
    blkio   4       121     1
    memory  8       137     1
    devices 7       121     1
    freezer 3       16      1
    net_cls 6       16      1

Should have *1* in enabled columns for cpu and memory.

## External disk support

### Install exfat-nofuse driver

    # Install exfat-nofuse from source
    sudo apt-get install build-essential
    sudo apt-get install raspberrypi-kernel-headers
    git clone https://github.com/cryptomilk/exfat-nofuse.git
    sudo apt-get install dkms
    sudo mkdir /usr/src/exfat-1.2.8
    cd exfat-nofuse
    sudo cp -R * /usr/src/exfat-1.2.8
    sudo dkms add -m exfat -v 1.2.8
    sudo dkms build -m exfat -v 1.2.8
    sudo dkms install -m exfat -v 1.2.8

### Mount external disks

    #Find disk UUID
    sudo blkid

    # Edit /etc/fstab - Use your disk UUID
    # exfat
    UUID=59A1-F35F /mnt/1TB-WDred exfat nofail,noatime,nodiratime,rw,async,umask=0,defaults,uid=1000,gid=1000 0 0
    # ext4
    UUID=AAAA-AAAA  /mnt/xyz  ext4  defaults,noatime,nodiratime,data=writeback 0 1

## Disable Swap

    sudo dphys-swapfile swapoff
    sudo dphys-swapfile uninstall
    sudo update-rc.d dphys-swapfile remove

-------------------------------------------------------------------------------

## Install Fail2ban

Block external IP login attempts via SSH

    sudo apt-get update
    sudo apt-get install fail2ban
    sudo vi /etc/fail2ban/jail.local

    [ssh]
    enabled = true
    port = ssh
    filter = sshd
    logpath = /var/log/auth.log
    bantime = 900
    banaction = iptables-allports
    findtime = 900
    maxretry = 3

    sudo systemctl restart fail2ban

-------------------------------------------------------------------------------

## Install Docker

    sudo apt-get update && sudo apt-get upgrade
    curl -sSL https://get.docker.com | sh
    sudo usermod -aG docker pi

Install docker-compose

    apt-get install -qy python-pip --no-install-recommends
    pip install pip --upgrade
    pip install docker-compose
    pip install --upgrade --force-reinstall docker-compose

## Monitoring Container Infrastructure

Create following entries on DNS for the Ingress Controller:

| Record |   Host  |        Value        |
| :----- | :------ | :------------------ |
| A      | cloud   | Your IP/DynDNS      |
| CNAME  | *.cloud | cloud.mydomain.com. |

Watch the dot in the end on CNAME record value.

Enable port-forwarding on your router from ports 80 and 443 to your internal IP address.

### Automated deployment

    git clone https://github.com/carlosedp/rpi-monitoring.git
    cd rpi-monitoring

To launch the Traefik ingress and portainer use:

    docker-compose -f traefik-portainer.yml up -d

To launch the monitoring suite (node_explorer/Prometheus/cAdvisor/Grafana)

    docker-compose up -d

### Manual deployment:

    docker network create monitoring

#### Traefik
Ingress Controller with SSL termination

Config - `traefik.toml`

```
defaultEntryPoints = ["http","https"]
debug = false
logLevel = "INFO"

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]

[acme]
email = "myemail@domain.com"
storage = "/etc/traefik/acme.json"
entryPoint = "https"
onDemand = false
OnHostRule = true
caServer = "https://acme-v01.api.letsencrypt.org/directory"
[acme.httpChallenge]
entryPoint="http"

# Web configuration backend
[web]
  address = ":8080"

[web.metrics.prometheus]
  buckets=[0.1,0.3,1.2,5.0]
  entryPoint = "traefik"

# Docker configuration backend
[docker]
  domain = "cloud.mydomain.com"
  watch = true
  exposedbydefault = false
```

Run the container

    docker run -d \
      --name=traefik \
      --restart=unless-stopped \
      --net=monitoring \
      -p 8080:8080 \
      -p 80:80 \
      -p 443:443 \
      -v $PWD/:/etc/traefik/ \
      -v /var/run/docker.sock:/var/run/docker.sock \
      traefik:v1.5.0-rc5 \
      --configFile=/etc/traefik/traefik.toml \
      --logLevel=DEBUG

#### Portainer in Docker

    docker volume create portainer_data
    docker run -d \
      --name portainer \
      --restart always \
      --net=monitoring \
      -p 9000:9000 \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v portainer_data:/data \
      --label "traefik.backend=portainer" \
      --label "traefik.frontend.rule=Host:portainer.cloud.mydomain.com" \
      --label "traefik.docker.network=monitoring" \
      --label "traefik.enable=true" \
      --label "traefik.port=9000" \
      --label "traefik.default.protocol=http" \
      portainer/portainer

#### node_exporter in Docker:

    docker run -d \
      --name=node_exporter \
      --restart always \
      --net=monitoring \
      -p 9100:9100 \
      -v "/proc:/host/proc" \
      -v "/sys:/host/sys" \
      -v "/:/rootfs" \
      napnap75/rpi-prometheus:node_exporter \
      --path.procfs=/host/proc \
      --path.sysfs=/host/sys
      --collector.filesystem.ignored-mount-points \
      "^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)"

Manual install instructions for running node_exporter on the host. **Not needed with it running on a container:**

    https://github.com/prometheus/node_exporter/releases/download/v0.15.2/node_exporter-0.15.2.linux-armv7.tar.gz

Unzip to `/usr/local/bin`

Create: `/etc/node_exporter`

    OPTIONS="--collector.textfile.directory /var/lib/node_exporter/textfile_collector"

Create: `/lib/systemd/system/node_exporter.service`
```
    [Unit]
    Description=Prometheus node exporter
    After=local-fs.target network-online.target network.target
    Wants=local-fs.target network-online.target network.target

    [Service]
    EnvironmentFile=/etc/node_exporter
    ExecStart=/usr/local/bin/node_exporter $OPTIONS
    Type=simple

    [Install]
    WantedBy=multi-user.target
```
Start service:

    sudo systemctl daemon-reload
    sudo systemctl start node_export

### cAdvisor
Docker monitoring agent

        sudo docker run -d \
          --name=cadvisor \
          --restart=unless-stopped \
          --net=monitoring \
          --volume=/:/rootfs:ro \
          --volume=/var/run:/var/run:rw \
          --volume=/sys:/sys:ro \
          --volume=/var/lib/docker/:/var/lib/docker:ro \
          --publish=8082:8080 \
          carlosedp/rpi-cadvisor


#### Prometheus config
Create `/etc/prometheus.yml`

```
# my global config
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # By default, scrape targets every 15 seconds.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'rpi-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - 'alert.rules'

# A scrape configuration containing exactly one endpoint to scrape:

scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 15s
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'node'
    scrape_interval: 10s
    #static_configs:
    #  - targets: ['192.168.1.4:9100']
    dns_sd_configs:
      - names:
        - 'node_exporter'
        type: 'A'
        port: 9100
  - job_name: 'cadvisor'
    scrape_interval: 10s
    #static_configs:
    #  - targets: ['cadvisor:8080']
    dns_sd_configs:
      - names:
        - 'cadvisor'
        type: 'A'
        port: 8080
  - job_name: 'traefik'
    scrape_interval: 10s
    #static_configs:
    #  - targets: ['traefik:8080']
    dns_sd_configs:
      - names:
        - 'traefik'
        type: 'A'
        port: 8080
```

#### Prometheus

    docker volume create prometheus_data

    docker run -d \
      --name=prometheus \
      --restart=unless-stopped \
      --net=monitoring \
      -p 9090:9090 \
      -v /etc/prometheus.yml:/etc/prometheus/prometheus.yml \
      -v prometheus_data:/prometheus \
      napnap75/rpi-prometheus:prometheus \
      --config.file=/etc/prometheus/prometheus.yml \
      --storage.tsdb.path=/prometheus \
      --storage.tsdb.retention=10d \
      --web.console.libraries=/usr/share/prometheus/console_libraries \
      --web.console.templates=/usr/share/prometheus/consoles

#### Grafana
      docker volume create grafana_data
      docker run -d \
        --name=grafana \
        --restart=unless-stopped \
        --net=monitoring \
        -e "GF_SERVER_ROOT_URL=http://192.168.1.4:3000" \
        -v grafana_data:/var/lib/grafana \
        -p 3000:3000 \
        --label "traefik.backend=grafana" \
        --label "traefik.frontend.rule=Host:grafana.cloud.mydomain.com" \
        --label "traefik.docker.network=monitoring" \
        --label "traefik.enable=true" \
        --label "traefik.port=3000" \
        --label "traefik.default.protocol=http" \
        fg2it/grafana-armhf:v4.6.3

Configure Prometheus datasource and install Grafana Dashboards:

* 179
* 893
* 1860
* 2870

-------------------------------------------------------------------------------

### DDClient in Docker
Dynamic DNS client

    mkdir -p /opt/ddclient

    #Create /etc/ddclient.conf
    tee  /etc/ddclient.conf << EOF
    use=web, web=dynamicdns.park-your-domain.com/getip
    protocol=namecheap
    server=dynamicdns.park-your-domain.com
    login=mydomain.com
    password=xxxxxxxxx
    home
    EOF

    docker run -d \
      --name=ddclient \
      --restart always \
      -v "/etc/ddclient.conf:/etc/ddclient/ddclient.conf" \
      kacis/docker-rpi-ddclient

-------------------------------------------------------------------------------

## Media Server

### Media Folder Structure

    /mnt/1TB-WDRed/Downloads/
                            /Incomplete
                            /Movies
                            /TVShows
                  /Movies/
                  /TVShows/
                  /Concerts/

The containers expect a similar structure with Downloads, Incomplete, Movies, TVShows and Concerts directories on your drive. Adjust the environment variables or .env file accordingly.

To make all applications behave similarly, the external volume will be mounted on `/volumes/media`.

Setup port-forwarding on your router to ports 32400 (for Plex) and 51413 (for transmission) to your internal IP where the server runs.

Automated deployment:

    git clone https://github.com/carlosedp/rpi-media-server.git
    cd rpi-media-server
    docker-compose up -d

Manual deployment:

### Environment variables

    export MEDIA=/mnt/1TB-WDred

### Transmission
Torrent Client

    docker volume create transmission_config
    docker run -d \
      --name transmission \
      --restart=unless-stopped \
      --net=mediaserver \
      -p  9091:9091 \
      -p  51413:51413 \
      -p  51413:51413/udp \
      -v $MEDIA:/volumes/media \
      -v transmission_config:/etc/transmission-daemon \
      -v /etc/localtime:/etc/localtime:ro \
      carlosedp/rpi-transmission

### SickRage
TV Show downloader

    docker volume create sickrage_config
    docker run -d \
    --name=sickrage \
    --restart=unless-stopped \
    --net=mediaserver \
    -v sickrage_config:/config \
    -v $MEDIA:/volumes/media \
    -v /etc/localtime:/etc/localtime:ro \
    -e PGID=1000 -e PUID=1000  \
    -p 8081:8081 \
    carlosedp/rpi-sickrage

### CoachPotato
Movie downloader

    docker volume create couchpotato_config
    docker volume create couchpotato_data
    docker run -d \
    --name couchpotato \
    --restart=unless-stopped \
    --net=mediaserver \
    -p 5050:5050 \
    -v $MEDIA:/volumes/media \
    -v couchpotato_data:/volumes/data \
    -v couchpotato_config:/volumes/config \
    -v /etc/localtime:/etc/localtime:ro \
    carlosedp/rpi-couchpotato

### Plex
Media Server

    docker volume create plex_config
    docker run -d \
      --name plex \
      --restart=unless-stopped \
      --net=mediaserver \
      -v $MEDIA:/media \
      -v plex_config:/root/Library \
      -v /etc/localtime:/etc/localtime:ro \
      -p 32400:32400/tcp \
      -p 3005:3005/tcp \
      -p 8324:8324/tcp \
      -p 32469:32469/tcp \
      -p 1900:1900/udp \
      -p 32410:32410/udp \
      -p 32412:32412/udp \
      -p 32413:32413/udp \
      -p 32414:32414/udp \
      jaymoulin/plex

### VNStat
Collects network interface stats

    docker volume create vnstat_data
    docker run -d \
      --name vnstatd \
      --restart=unless-stopped \
      --privileged=true \
      --net=mediaserver \
      -v vnstat_data:/var/lib/vnstat \
      carlosedp/rpi-vnstatd

### HTPCManager
Manage the Media Server services

    docker volume create htpcmanager_config
    docker run -d \
      --name=htpcmanager \
      --restart=unless-stopped \
      --net=mediaserver \
      --net=monitoring_default \
      -p 8085:8085 \
      -v htpcmanager_config:/config \
      -v vnstat_data:/var/lib/vnstat \
      -v /mnt:/mnt \
      -v /etc/localtime:/etc/localtime:ro \
      -e PGID=1000 -e PUID=1000 \
      --label "traefik.backend=htpcmanager" \
      --label "traefik.frontend.rule=Host:htpcmanager.cloud.mydomain.com" \
      --label "traefik.enable=true" \
      --label "traefik.port=8085" \
      --label "traefik.docker.network=monitoring_default" \
      --label "traefik.default.protocol=http"


      carlosedp/rpi-htpcmanager

To make Traefik forward traffic to the Media Server Manager, connect it to the monitoring network (check monitoring network with `docker network ls`):

    docker network connect monitoring_default htpcmanager


-------------------------------------------------------------------------------

## Misc Containers

### Airsonos
Expose Sonos over Airplay via a Docker RPi container

    docker run -d \
      --name="airsonos" \
      --net="host" \
      --restart=unless-stopped \
      -p 5000-5015:5000-5015/tcp \
      fstehle/rpi-airsonos

### Minecraft
    docker run -d=true -p=25565:25565 --name mc -e _JAVA_OPTIONS='-Xms512m -Xmx512m' -v=/mnt/minecraft:/data kroonwijk/rpi-minecraft /start

-------------------------------------------------------------------------------

## Docker Volume Backup/Restore

### backup.sh [volume name]
    #!/bin/bash

    if [[ $1 == '-a' ]]; then
        VOLUMES=$(docker volume ls -q)
    elif [[ $1 ]]; then
        if `docker volume ls -q |grep -w $1` ; then
            VOLUMES=$1
        else
          echo "Volume $1 does not exist."
          exit
        fi
    else
        echo "No parameter passed, provide a volume name or '-a' to backup all volumes."
        exit
    fi

    for i in $VOLUMES; do
        echo Backup volume: $i
        export DOCKER_VOLUME=$i
        docker run -d \
          --name=backup-$i \
          --rm \
          -v $DOCKER_VOLUME:/$DOCKER_VOLUME:ro \
          -v /mnt/1TB-WDred/Backups/:/backup \
          alpine \
          tar -czpf /backup/$DOCKER_VOLUME-$(date +%Y%m%d_%H%M%S).tar.gz $DOCKER_VOLUME
    done


### restore.sh [/path/backup_file-20001212_000000.tar.gz]

        #!/bin/bash
        pathToFile=$(dirname $1)
        fileName=$(basename $1)
        VOLUME=$(echo $fileName | sed 's/\(.*\)-[0-9]*_[0-9]*\.tar\.gz/\1/')
        # Check if docker volume exists
        if `docker volume ls -q |grep $VOLUME` ; then
            read -p "Volume already exists, overwrite all files? " -n 1 -r
            echo    # (optional) move to a new line
            if [[ ! $REPLY =~ ^[Yy]$ ]]
            then
                [[ "$0" = "$BASH_SOURCE" ]] && exit 1 || return 1
            fi
        else
            echo "Creating volume $VOLUME"
            docker volume create $VOLUME
        fi
        exit
        echo Restore volume: $VOLUME
        docker run -d \
          --name=restore-$VOLUME \
          --rm \
          -v $pathToFile:/source:ro \
          -v $VOLUME:/$VOLUME \
          alpine \
          tar zvxf -C /$VOLUME $pathToFile/$fileName --strip-components 1
