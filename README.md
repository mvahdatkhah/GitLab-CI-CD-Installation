#  GitLab CI/CD Installation via Docker Compose (Ubuntu 24.04)

---

##  Prerequisites

Run everything as a non-root `sudo` user.

### 1. Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Docker and Docker Compose
```bash
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \

sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine and Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

##  Docker Post-Installation Steps (Recommended)

After installing Docker, apply these optional (but helpful) steps:

### 1. Run Docker without `sudo`
```bash
sudo usermod -aG docker $USER
newgrp docker
```

Log out and back in, or reboot, to apply the group change.

### 2. Configure Docker to start on boot
```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

### 3. Test your Docker setup
```bash
docker run hello-world
```

##  Project Structure

```bash
gitlab-docker/
 docker-compose.yml
 README.md
```

---

##  Create `docker-compose.yml`

```yaml
version: '3.8'

services:
  gitlab:
    image: gitlab/gitlab-ce:17.11.0-ce.0
    container_name: gitlab
    restart: always
    hostname: 'gitlab.example.com'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.example.com'
        letsencrypt['enable'] = true
        letsencrypt['contact_emails'] = ['admin@example.com']
        nginx['redirect_http_to_https'] = true
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
        gitlab_rails['time_zone'] = 'Tehran'
    ports:
      - '80:80'
      - '443:443'
      - '2224:22'
    volumes:
      - './config:/etc/gitlab'
      - './logs:/var/log/gitlab'
      - './data:/var/opt/gitlab'
      - './backups:/var/opt/gitlab/backups'
    networks:
      - gitlab-network    

networks:
  gitlab-network:
    driver: bridge

```

Replace `gitlab.local` with your server IP or FQDN.

---

##  Start GitLab

###  Pull GitLab Image
```bash
docker compose pull
```

###  Start the container
```bash
docker compose up -d
```

Monitor logs with:

```bash
docker logs -f gitlab
```

---

##  Access GitLab

- Navigate to: `http://your-server-ip/`
- First login will ask you to set the root password.

---

##  Open Firewall Ports (with `iptables`)

```bash
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # HTTP
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # HTTPS
sudo iptables -A INPUT -p tcp --dport 2224 -j ACCEPT  # SSH for GitLab
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
```

### Save iptables rules across reboots:

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

---

##  GitLab Backups

###  Backup command

Inside the container:

```bash
docker exec -it gitlab gitlab-rake gitlab:backup:create
```

Backup files are saved in: `./backups/` (host) or `/var/opt/gitlab/backups` (container)

###  Enable automatic daily backups at 2AM

```bash
docker exec -it gitlab bash
crontab -e
```

Add:

```cron
0 2 * * * /opt/gitlab/bin/gitlab-rake gitlab:backup:create CRON=1
```

###  Keep backups for 7 days

Edit `./config/gitlab.rb` or inside container:

```ruby
gitlab_rails['backup_keep_time'] = 604800
```

Then apply:

```bash
docker exec -it gitlab gitlab-ctl reconfigure
```

---

##  Restore from Backup

1. Stop GitLab:
```bash
docker compose down
```

2. Copy your backup file into `./backups/`

3. Restore:
```bash
docker compose up -d
docker exec -it gitlab gitlab-backup restore BACKUP=timestamp
```

---

##  Notes

- GitLab config is stored in `./config`
- All CI/CD features included
- HTTPS can be added later
- GitLab Runner can be added via Docker

---
