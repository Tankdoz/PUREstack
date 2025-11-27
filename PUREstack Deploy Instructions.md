# P.U.R.E.---> (P)i-hole, (U)nbound, (R)edis, and (E)xporters
---
this stack deploys a full-featured DNS solution with ad-blocking, 
recursive DNS resolution, caching, metrics exporting and log collection  

  - unbound log collection can be pushed to Loki via Promtail
  - unbound stats can be scraped by Prometheus
  - Prometheus and Loki can be used as data sources in Grafana for visualization
  
For more details, see the PUREstack project page: https://github.com/Virgil-TD/PUREstack  
A stack for grafana, prometheus and loki is available here: https://github.com/Virgil-TD/GRAPLstack  
For a grafana dashboard for unbound metrics see: https://github.com/ar51an/unbound-dashboard  

These are instructions on how to deploy PUREstack on a new host with Ubuntu 24.04 or later or Debian 13 or later  
**Pay attention to the volume directory permissions as different containers can run under different UIDs and GIDs**  
**the permissions are set accordingly and explained in the relevant sections below**  
**the possibility to edit configuration files as your user via ssh or vscode is preserved**  

## Prerequisites:  
- a host running either Ubuntu 24.04 or Debian 13 is preferred, older versions may work as well
- a user with sudo rights
- these instructions assume an installation in ~/dockerprojects/stacks/pure, change to your own preferences
### images:
- pihole/pihole:latest  
- klutchell/unbound:latest
- redis:latest
- grafana/promtail:3.5.8 (latest may work as well)
- unbound-exporter (build yourself from provided dockerfile)


## Step 1: Update and upgrade the system
```
sudo apt update && sudo apt upgrade -y
sudo reboot
```

## Step 2: Install Docker and Docker Compose  
You can skip this if already installed or use the Docker-approved method as described in Official Docker Install.md

```
sudo apt install -y docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
````

## Step 3: Create the necessary directories and permissions  
Clone PUREstack so you get the right files with the right directory structure; permissions will be updated later
```
git clone https://github.com/Virgil-TD/PUREstack.git \
~/dockerprojects/stacks/pure
```
### 3a. Pi-hole volume directories and permissions  
- Pi-hole runs as root inside the container
- It can create and manage its own volume directories without manual chown or mkdir
- Configuration is normally handled via the Pi-hole web GUI
- If you want to enforce specific settings, define them in docker-compose or an additional .env file

### 3b. Unbound volume directories and permissions
- The klutchel/unbounf image is distroless --> contains only the necessary binaries and libraries to run Unbound, no shell or package manager
- The image runs Unbound as a non-privileged user UID 101, GID 102 after startup
- Configuration file unbound.conf and aditional configuration files *,conf in unbound.conf.d need to be readable by 101:102
- blocklist, although not used by unbound (pihole is doing the blocking) needs to be there to avoid warnings from the exporter

unbound.conf: rw for your user, read for 101:102  
unbound.conf.d: rw for your user, read for 101:102  
blocklists: editable by you, readable by 101:102  
```
sudo chown $USER:102 ~/dockerprojects/stacks/pure/volumes/unbound/unbound.conf
sudo chmod 640 ~/dockerprojects/stacks/pure/volumes/unbound/unbound.conf
sudo chown -R $USER:102 ~/dockerprojects/stacks/pure/volumes/unbound/unbound.conf.d
sudo chmod -R 640 ~/dockerprojects/stacks/pure/volumes/unbound/unbound.conf.d
sudo chown -R $USER:102 ~/dockerprojects/stacks/pure/volumes/unbound/blocklists
sudo chmod -R 640 ~/dockerprojects/stacks/pure/volumes/unbound/blocklists
```

### 3c. Redis volume directories, files and permissions
- Redis runs as non-privileged user UID 999, GID 999 inside the container
- The data directory must be writable by 999:999
- the volume mapping ensures persistence of Redis data across container restarts
- there is no need to make the directory writable by your user
```
mkdir -p ~/dockerprojects/stacks/pure/volumes/redis/data
sudo chown -R 999:999 ~/dockerprojects/stacks/pure/volumes/redis/data
```

### 3d. Unbound-exporter directories, files and permissions
---
- there is no supported image for https://github.com/ar51an/unbound-exporter
- you need to build it locally from the provided Dockerfile in ~/dockerprojects/stacks/pure/exporter
- Unbound-exporter runs as non-privileged user UID 1000, GID 1000 inside the container
- It needs read access to the Unbound control socket and the blocklists directory
- The control socket is created by Unbound in the /run/unbound directory which is mapped via a named volume (compose.yaml)
- The blocklists directory is mapped from the host system
- No special permission changes are needed here as the Unbound volume directories are already set above

build the unbound-exporter image locally, it will be availabe in the image list (docker images) as unbound-exporter:latest
```
cd ~/dockerprojects/stacks/pure/exporter
docker build -t unbound-exporter:latest .
```

### 3e. Promtail volume directories and permissions
- Promtail runs as non-privileged user UID 1000, GID 1000 inside the container
- The promtail volume directory must be writable by 1000:1000

positions.yaml:  
create positions.yaml to avoid errors on first startup, owned by promtail user 1000:1000 and only readable by own user
```
sudo touch ~/dockerprojects/stacks/pure/volumes/promtail/positions.yaml
sudo chown 1000:1000 ~/dockerprojects/stacks/pure/volumes/promtail/positions.yaml
sudo chmod 640 ~/dockerprojects/stacks/pure/volumes/promtail/positions.yaml
```
promtail-config.yaml:  
owned by your user and readable by promtail user 1000:1000
```
sudo chown $USER:1000 ~/dockerprojects/stacks/pure/volumes/promtail/promtail-config.yaml
sudo chmod 660 ~/dockerprojects/stacks/pure/volumes/promtail/promtail-config.yaml
```

## Step 4: Start your PURE stack(s):
Multiple PURE stacks can be started in case you want multiple DNS resolvers. It's not possible though, to deploy multiple stacks on the same host, as a stack needs unique ownership of port 53. In order to differentiate between services of different PUREstacks, they have to be started with a unique name since they share the same monitoring network. This is aligned with the scraping configuration of Prometheus in the GRAPLstack https://github.com/Virgil-TD/GRAPLstack

```
cd ~/dockerprojects/stacks/pure
# first stack to be deployed with
docker compose -p pure1 up -d
```
```
# second stack to be deployed with
docker compose -p pure2 up -d
```
```
# third stack to be deployed with 
docker compose -p pure3 up -d
```