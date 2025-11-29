Official Docker CE (Community Edition) installation method:
---
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

# activat new group permissions (logoff/logon after is required)
newgrp docker
```
log off and log on to finalize official docker installation
