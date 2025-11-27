P.U.R.E.---> (P)i-hole, (U)nbound, (R)edis, and (E)xporters
---
this stack deploys a full-featured DNS solution with ad-blocking, 
recursive DNS resolution, caching, metrics exporting and log collection  

  - unbound log collection can be pushed to Loki via Promtail
  - unbound stats can be scraped by Prometheus
  - Prometheus and Loki can be used as data sources in Grafana for visualization
  
For more details, see the PUREstack project page: https://github.com/yourusername/PUREstack  
A stack for grafana, prometheus and loki is available here: https://github.com/yourusername/GRAPLstack  
For a grafana dashboard for unbound metrics see: https://github.com/ar51an/unbound-dashboard  

These are instructions on how to deploy PUREstack on a new host with Ubuntu 24.04 or later or Debian 13 or later  
**Pay attention to the volume directory permissions as different containers can run under different UIDs and GIDs**  
**the permissions are set accordingly and explained in the relevant sections below**  
**the possibility to edit configuration files as your user via ssh or vscode is preserved**  

Prerequisites:  
---
- a host running either Ubuntu 24.04 or Debian 13 is preferred, older versions may work as well
images:
- pihole/pihole:latest  
- klutchell/unbound:latest
- redis:latest
- promtail:3.5.8
- exporter (build yourself from provided dockerfile)

environment variables:  
---
```
# if you want to copy/paste the codeblocks, be sure to copy the environment variables first
# you can change them for your own setup

DOCKERPROJECTSDIR=~/dockerprojects        #  Preferred location for your dockerprojects folder, default is /dockerprojects under user home directory  
STACKSDIR=/stacks                         #  Preferred location for your stacks folder under dockerprojects, default is /stacks  
PUREDIR=/pure                             #  Preferred location for your PUREstack directory, default is /pure  
WEBPASSWORD=changeme                      #  Change this to your desired Pi-hole web interface password  
```

Step 1: Update and upgrade the system
---
```
sudo apt update && sudo apt upgrade -y
sudo reboot
```

Step 2: Install Docker and Docker Compose  
---
You can skip this if already installed or use the easy or the Docker-approved method below  

Easy method to instal Docker
```
sudo apt install -y docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
````

Official Docker CE (Community Edition) installation method:
```
# remove old versions if any
sudo apt remove docker docker-engine docker.io containerd runc -y

# install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# add Docker’s official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# add Docker’s official repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# install Docker Engine, CLI and Containerd
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# verify installation
sudo docker run hello-world

# add your user to the docker group to run docker without sudo
sudo usermod -aG docker $USER
newgrp docker   #activates new group permissions, but logoff,logon will be required
```
log off and log on to finalize official docker installation


Step 3: Create the necessary directories and permissions  
---

  3a. Pi-hole volume directories, files and permissions  
---
- Pi-hole runs as root inside the container
- It can create and manage its own volume directories without manual chown or mkdir
- Configuration is normally handled via the Pi-hole web GUI
- If you want to enforce specific settings, define them in docker-compose (e.g. environment variables)
- For more details, refer to the official Pi-hole Docker documentation: https://docs.pi-hole.net/hosting/docker/

  3b. Unbound volume directories, files and permissions
---
- The klutchel/unbounf image is distroless --> contains only the necessary binaries and libraries to run Unbound, no shell or package manager
- The image runs Unbound as a non-privileged user UID 101, GID 102 after startup
- Configuration file unbound.conf mounted in /etc/unbound and aditional configuration files *,conf in /etc/unbound/unbound.conf.d need to be readable by 101:102

these directories used in the compose.yaml must be created manually before first startup
```
mkdir -p ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound
mkdir -p ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/unbound.conf.d
mkdir -p ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/blocklists
mkdir -p ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/logs
```

copy your unbound.conf and additional configuration files into the relevant directories here before setting permissions
```
wget -P ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/unbound.conf https://raw.githubusercontent.com/Virgil-TD/PUREstack/unbound.conf 
wget -P ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/unbound.conf.d/cachedb.conf /https://raw.githubusercontent.com/Virgil-TD/PUREstack/unbound.conf.d/cachedb.conf
wget -P ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/unbound.conf.d/logging.conf /https://raw.githubusercontent.com/Virgil-TD/PUREstack/unbound.conf.d/logging.conf
wget -P ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/unbound.conf.d/remote-control.conf /https://raw.githubusercontent.com/Virgil-TD/PUREstack/unbound.conf.d/remote-control.conf
```

although in the PURE-setup Pihole manages blocking via its own blocklists, the Unbound-exporter metrics require a blocklist to be present
create a default blocklist file to avoid errors if no blocklists are added yet
```
cat <<EOF > ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/blocklists/default-blocklist.conf
local-zone: "ads.example.com." always_nxdomain
local-zone: "tracking.example.org." always_null
local-zone: "malware.testsite.net." always_refuse
EOF
```
create a default empty log file to avoid errors if logging to file is enabled (optional)
```
touch ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/logs/unbound.log
```
unbound.conf: rw for your user, read for 101:102
```
sudo chown $USER:102 ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/unbound.conf
sudo chmod 640 ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/unbound.conf
```
unbound.conf.d: rw for your user, read for 101:102
```
sudo chown -R $USER:102 ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/unbound.conf.d
sudo chmod -R 640 ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/unbound.conf.d
```
blocklists: editable by you, readable by 101:102
```
sudo chown -R $USER:102 ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/blocklists
sudo chmod -R 640 ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/blocklists
```
logs: must be writable by Unbound (101:102), not by you
```
sudo chown -R 101:102 ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/logs
sudo chmod -R 750 ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/unbound/logs
```

  3c. Redis volume directories, files and permissions
---
- Redis runs as non-privileged user UID 999, GID 999 inside the container
- The data directory must be writable by 999:999
- the volume mapping ensures persistence of Redis data across container restarts
- there is no need to make the directory writable by your user
```
mkdir -p ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/redis/data
sudo chown -R 999:999 ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/redis/data
```

  3d. Unbound-exporter directories, files and permissions
---
- there is no supported image for Unbound-exporter;
- you need to build it locally from the provided Dockerfile that you can place in the dockerprojects/stacks/pure/exporter directory
- Unbound-exporter runs as non-privileged user UID 1000, GID 1000 inside the container
- It needs read access to the Unbound control socket and the blocklists directory
- The control socket is created by Unbound in the /run/unbound directory which is mapped via a named volume
- The blocklists directory is mapped from the host system
- No special permission changes are needed here as the Unbound volume directories are already set above

create the directory for the Dockerfile if it does not exist
```
mkdir -p ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/exporter
```
copy the Dockerfile for unbound-exporter to the relevant directory
```
wget -P ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/volumes/exporter/Dockerfile https://raw.githubusercontent.com/Virgil-TD/PUREstack/exporter/Dockerfile
```
build the unbound-exporter image locally, it will be availabe in the image list (docker images) as unbound-exporter:latest
```
cd ${DOCKERPROJECTS}${STACKSDIR}${PUREDIR}/exporter
docker build -t unbound-exporter:latest .
```

  3 e. Promtail volume directories and permissions
---
- Promtail runs as non-privileged user UID 1000, GID 1000 inside the container
- The promtail volume directory must be writable by 1000:1000

create promtail volume directory
```
mkdir -p ${DOCKERPROJECTSDIR}${STACKSDIR}${PUREDIR}/volumes/promtail
```
create positions.yaml to avoid errors on first startup, owned by promtail user 1000:1000 and only readable by own user
```
sudo touch ${DOCKERPROJECTSDIR}${STACKSDIR}${PUREDIR}/volumes/promtail/positions.yaml
sudo chown 1000:1000 ${DOCKERPROJECTSDIR}${STACKSDIR}${PUREDIR}/volumes/promtail/positions.yaml
sudo chmod 640 ${DOCKERPROJECTSDIR}${STACKSDIR}${PUREDIR}/volumes/promtail/positions.yaml
```
promtail-config.yaml: created empty here, you need to fill it with your configuration later 
owned by your user and readable by promtail user 1000:1000
```
sudo touch ${DOCKERPROJECTSDIR}${STACKSDIR}${PUREDIR}/volumes/promtail/promtail-config.yaml
sudo chown $USER:1000 ${DOCKERPROJECTSDIR}${STACKSDIR}${PUREDIR}/volumes/promtail/promtail-config.yaml
sudo chmod 660 ${DOCKERPROJECTSDIR}${STACKSDIR}${PUREDIR}/volumes/promtail/promtail-config.yaml
```

Step 4: Download the PUREstack Docker Compose file
---
```
cd ${DOCKERPROJECTSDIR}${STACKSDIR}${PUREDIR}
wget https://raw.githubusercontent.com/Virgil-TD/PUREstack/compose.yaml
```


