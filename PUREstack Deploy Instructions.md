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
the usernames may look odd on the host but the ID's are correct

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
Clone PUREstack so you get the right files with the right directory structure; permissions will be updated
```
git clone https://github.com/Virgil-TD/PUREstack.git \
~/dockerprojects/stacks/pure

sudo chown $USER:102 ~/dockerprojects/stacks/pure/volumes/unbound/unbound.conf
sudo chmod 640 ~/dockerprojects/stacks/pure/volumes/unbound/unbound.conf
sudo chown -R $USER:102 ~/dockerprojects/stacks/pure/volumes/unbound/unbound.conf.d
sudo chmod 750 ~/dockerprojects/stacks/pure/volumes/unbound/unbound.conf.d
sudo chown -R $USER:102 ~/dockerprojects/stacks/pure/volumes/unbound/blocklists
sudo chmod 750 ~/dockerprojects/stacks/pure/volumes/unbound/blocklists
sudo chmod 640 ~/dockerprojects/stacks/pure/volumes/unbound/blocklists/unbound.block.conf

mkdir -p ~/dockerprojects/stacks/pure/volumes/redis/data
sudo chown -R 999:999 ~/dockerprojects/stacks/pure/volumes/redis/data
sudo chmod 770 ~/dockerprojects/stacks/pure/volumes/redis/data

cd ~/dockerprojects/stacks/pure/exporter
docker build -t unbound-exporter:latest .

sudo touch ~/dockerprojects/stacks/pure/volumes/promtail/positions.yaml
sudo chown 1000:1000 ~/dockerprojects/stacks/pure/volumes/promtail/positions.yaml
sudo chmod 640 ~/dockerprojects/stacks/pure/volumes/promtail/positions.yaml
sudo chown $USER:1000 ~/dockerprojects/stacks/pure/volumes/promtail/promtail-config.yaml
sudo chmod 660 ~/dockerprojects/stacks/pure/volumes/promtail/promtail-config.yaml
```

## Step 4: Start your PURE stack(s):
Multiple PURE stacks can be started in case you want multiple DNS resolvers. It's not possible though, to deploy multiple stacks on the same host, as a stack needs unique ownership of port 53. 
In order to differentiate between the different stacks you should use the .env file. For the PURE stack the default .env file is:

.env file:  
```
cd ~/dockerprojects/stacks/pure
cat > .env <<EOF
WEBPASSWORD=changeme     # choose your own
HOSTNAME=PUREstack-010   # this name will be used by promtail to label the data from this stack, reuse it in your prometheus configuration 
LOKIIP=192.168.1.22      # IP adress of your LOKI server
EOF
```
edit the .env file to reflect the settings in your environment
```
sudo nano .env
```

next bring up your stack
```
cd ~/dockerprojects/stacks/pure
docker compose up -d
```
## Step 5: Log in to Pihole and set Unbound as its upstream DNS resolver
- Open a webbrowser and enter ip-adress-pihole/admin  (use your ip-adress)
- you will get a logon screen where you can enter the password that you set in your .env file
- go to Settings > DNS : untick all upstream DNS resolvers and enter 127.0.0.1#5335 in custom DNS providers
- this will ensure Pihole forwards only to Unbound

