# 🦊 GitLab CI/CD Installation via Docker Compose (Ubuntu 24.04)

---

## ✅ Prerequisites

Run everything as a non-root `sudo` user.

### 🔄 1. Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

### 🐳 2. Install Docker and Docker Compose
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
```

## ⚙️ 3. Docker Post-Installation Steps (Recommended)

After installing Docker, apply these optional (but helpful) steps:

### 👤 1. Run Docker without `sudo`
```bash
sudo usermod -aG docker $USER
newgrp docker
```

🔁 Log out and back in, or reboot, to apply the group change.

### 🔁 2. Configure Docker to start on boot
```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

### 🧪 3. Test your Docker setup
```bash
docker run hello-world
```

## 📂 Project Structure

```bash
/srv/gitlab-docker/
├── config     → GitLab configuration (/etc/gitlab)
├── data       → Persistent data (/var/opt/gitlab)
├── logs       → Logs mounted from LVM (/var/log/gitlab)
└── backups    → Backups mounted from LVM (/var/opt/gitlab/backups)
```

---

## 📝 Create `docker-compose.yml`

```yaml
---
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
        gitlab_rails['backup_path'] = '/var/opt/gitlab/backups'
    ports:
      - '80:80'
      - '443:443'
      - '2224:22'
    volumes:
      - '/srv/gitlab-docker/config:/etc/gitlab'
      - '/srv/gitlab-docker/logs:/var/log/gitlab'
      - '/srv/gitlab-docker/data:/var/opt/gitlab'
      - '/srv/gitlab-docker/backups:/var/opt/gitlab/backups'
    networks:
      - gitlab-network    

networks:
  gitlab-network:
    driver: bridge
...
```

🔁 Replace `gitlab.local` with your server IP or FQDN.

---

## 🏗️ Start GitLab

### 📥 Pull GitLab Image
```bash
docker compose pull
```

### ▶️ Start the container
```bash
docker compose up -d
```

### 📡 Monitor logs with:

```bash
docker logs -f gitlab
```

---

## 🌐 Access GitLab

- Navigate to: `http://your-server-ip/`
- 🔐 First login will ask you to set the root password.

---

## 🔐 Open Firewall Ports (with `iptables`)

```bash
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # HTTP
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # HTTPS
sudo iptables -A INPUT -p tcp --dport 2224 -j ACCEPT  # SSH for GitLab
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
```

### 💾 Save iptables rules across reboots:

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

---

## 🗂️ GitLab Backups

### 💡 Backup command (inside the container):

Inside the container:

```bash
docker exec -it gitlab gitlab-rake gitlab:backup:create
```
🗃️ Files saved to: `./backups/` (host) or `/var/opt/gitlab/backups` (container)

### ⏰ Enable automatic daily backups at 2AM

```bash
docker exec -it gitlab bash
crontab -e
```

Add:

```cron
0 2 * * * /opt/gitlab/bin/gitlab-rake gitlab:backup:create CRON=1
```

### 🧼 Keep backups for 7 days

Edit `./config/gitlab.rb` or inside container:

```ruby
gitlab_rails['backup_keep_time'] = 604800
```

Then apply:

```bash
docker exec -it gitlab gitlab-ctl reconfigure
```

---

## ♻️ Restore from Backup

1️⃣ Stop GitLab:
```bash
docker compose down
```

2️⃣ Copy your backup file into `./backups/`

3️⃣ Restore:
```bash
docker compose up -d
docker exec -it gitlab gitlab-backup restore BACKUP=timestamp
```

---

## 💽 Disk Setup with LVM

### 🧱 1. Create a physical volume
```bash
sudo pvcreate /dev/sdb
```

### 🧩 2. Create a volume group
```bash
sudo vgcreate gitlab-vg /dev/sdb
```

### 📦 3. Create logical volumes

```bash
# 150GB for logs
sudo lvcreate -L 150G -n gitlab-logs gitlab-vg

# Remaining (~50GB) for backups
sudo lvcreate -L 50G -n gitlab-backups gitlab-vg
```

### 📁 4. Format and mount

```bash
sudo mkfs.ext4 /dev/gitlab-vg/gitlab-logs
sudo mkfs.ext4 /dev/gitlab-vg/gitlab-backups

# Create mount points
sudo mkdir -p /mnt/gitlab/{logs,backups}

# Mount the volumes
sudo mount /dev/gitlab-vg/gitlab-logs /mnt/gitlab/logs
sudo mount /dev/gitlab-vg/gitlab-backups /mnt/gitlab/backups
```

### 📝 5. Persist the mounts in `/etc/fstab`
```bash
echo '/dev/gitlab-vg/gitlab-logs /mnt/gitlab/logs ext4 defaults 0 2' | sudo tee -a /etc/fstab
echo '/dev/gitlab-vg/gitlab-backups /mnt/gitlab/backups ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

🔁 Then remount or reboot: `sudo mount -a`

## 🔄 Bind-mount to Docker folders (optional)

If GitLab uses `./logs` and `./backups` in your project root:

```bash
sudo mkdir -p /srv/gitlab-docker/logs /srv/gitlab-docker/backups
sudo mount --bind /mnt/gitlab/logs /srv/gitlab-docker/logs
sudo mount --bind /mnt/gitlab/backups /srv/gitlab-docker/backups
```

Make these persistent via `/etc/fstab`:

```bash
echo '/mnt/gitlab/logs /srv/gitlab-docker/logs none bind 0 0' | sudo tee -a /etc/fstab
echo '/mnt/gitlab/backups /srv/gitlab-docker/backups none bind 0 0' | sudo tee -a /etc/fstab
```

## ⚙️ Setup LVM with Ansible

### 📁 Save as setup-lvm.yml
```yaml
---
- name: 🚀 Configure LVM and mount volumes for GitLab
  hosts: gitlab
  become: true

  vars:  # 🎯 Variable definitions
    disk_device: /dev/sdb
    vg_name: gitlab-vg
    lv_logs:
      name: gitlab-logs
      size: 150G
      mount: /mnt/gitlab/logs
    lv_backups:
      name: gitlab-backups
      size: 50G
      mount: /mnt/gitlab/backups
    filesystem_type: ext4

  tasks:

    - name: 📦 Install LVM dependencies
      tags: [setup, packages]
      apt:
        name: lvm2
        state: present
        update_cache: true

    - name: 🛠️ Create physical volume
      tags: [lvm, pv]
      command: pvcreate {{ disk_device }}
      args:
        creates: "/etc/lvm/archive"

    - name: 🧱 Create volume group
      tags: [lvm, vg]
      command: vgcreate {{ vg_name }} {{ disk_device }}
      args:
        creates: "/dev/{{ vg_name }}"

    - name: 🔧 Create logical volume for logs
      tags: [lvm, lv, logs]
      command: lvcreate -L {{ lv_logs.size }} -n {{ lv_logs.name }} {{ vg_name }}
      args:
        creates: "/dev/{{ vg_name }}/{{ lv_logs.name }}"

    - name: 🔐 Create logical volume for backups
      tags: [lvm, lv, backups]
      command: lvcreate -L {{ lv_backups.size }} -n {{ lv_backups.name }} {{ vg_name }}
      args:
        creates: "/dev/{{ vg_name }}/{{ lv_backups.name }}"

    - name: 🧼 Format logs volume
      tags: [format, logs]
      filesystem:
        fstype: "{{ filesystem_type }}"
        dev: "/dev/{{ vg_name }}/{{ lv_logs.name }}"

    - name: 🧼 Format backups volume
      tags: [format, backups]
      filesystem:
        fstype: "{{ filesystem_type }}"
        dev: "/dev/{{ vg_name }}/{{ lv_backups.name }}"

    - name: 📂 Create mount directories
      tags: [mount, directories]
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - "{{ lv_logs.mount }}"
        - "{{ lv_backups.mount }}"

    - name: 📎 Mount logs volume
      tags: [mount, logs]
      mount:
        path: "{{ lv_logs.mount }}"
        src: "/dev/{{ vg_name }}/{{ lv_logs.name }}"
        fstype: "{{ filesystem_type }}"
        opts: defaults
        state: mounted

    - name: 📎 Mount backups volume
      tags: [mount, backups]
      mount:
        path: "{{ lv_backups.mount }}"
        src: "/dev/{{ vg_name }}/{{ lv_backups.name }}"
        fstype: "{{ filesystem_type }}"
        opts: defaults
        state: mounted

    - name: 📝 Make logs mount persistent
      tags: [fstab, logs]
      mount:
        path: "{{ lv_logs.mount }}"
        src: "/dev/{{ vg_name }}/{{ lv_logs.name }}"
        fstype: "{{ filesystem_type }}"
        opts: defaults
        state: present

    - name: 📝 Make backups mount persistent
      tags: [fstab, backups]
      mount:
        path: "{{ lv_backups.mount }}"
        src: "/dev/{{ vg_name }}/{{ lv_backups.name }}"
        fstype: "{{ filesystem_type }}"
        opts: defaults
        state: present
...        
```

### 🛠️ Usage
Run the playbook from your Ansible control node:
```bash
ansible-playbook -i inventory setup-lvm.yml
```

Replace `inventory` with the path to your host or group inventory file. Target gitlab host/group accordingly.

## 🧹 Cron Job: Auto-Prune GitLab Backups

### 🗑️ Cleanup Script: /usr/local/bin/cleanup_gitlab_backups.sh

```bash
cat << 'EOF' | sudo tee /usr/local/bin/cleanup_gitlab_backups.sh
#!/bin/bash
BACKUP_DIR="/srv/gitlab-docker/backups"
MAX_BACKUPS=11

cd "$BACKUP_DIR" || exit 1
ls -1tr *.tar | head -n -$MAX_BACKUPS | while read -r oldfile; do
  echo "Removing $oldfile"
  rm -f "$oldfile"
done
EOF
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/cleanup_gitlab_backups.sh
```

### 📅 Add to cron (daily at 3AM)
```bash
(crontab -l; echo "0 3 * * * /usr/local/bin/cleanup_gitlab_backups.sh") | crontab -
```

## 🏃 GitLab Runner Installation (Docker, Remote Host)

### 1️⃣ Launch GitLab Runner Container
```bash
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

### 2️⃣ Register the Runner
Replace `https://gitlab.example.com` with your actual GitLab URL:
```bash
docker exec -it gitlab-runner gitlab-runner register
```

## 🔽 Fill in prompts using:

- 🌐 GitLab URL: `https://gitlab.example.com`
- 🔑 Token: Grab from GitLab Admin UI
- ⚙️ Executor: `docker`
- 🐳 Image: `alpine:latest`

You’ll be prompted interactively:

| Prompt               | Example                                |
|----------------------|----------------------------------------|
| GitLab instance URL  | https://gitlab.example.com             |
| Registration token   |  Get this from Admin > CI/CD > Runners |
| Description          | docker-remote-runner                   |
| Tags                 | docker,remote                          |
| Executor             | docker                                 |
| Default image        | alpine:latest                          |

Once registered, the runner will auto-connect to your GitLab instance and start executing jobs 🚀

## 🏃 GitLab Runner via Docker Compose (Remote Server)
### 1️⃣ 📁 Directory Setup

Create a dedicated folder to hold your docker-compose.yml and config:

```bash
mkdir -p /srv/gitlab-runner/config
cd /srv/gitlab-runner
```

### 2️⃣ 📝 Create docker-compose.yml
```bash
services:
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    volumes:
      - ./config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
```

This setup enables Docker-in-Docker for CI builds and stores config persistently.

### 🔐 Register the Runner
```bash
docker exec -it gitlab-runner gitlab-runner register
```

## 🔽 Fill in prompts using:

- 🌐 GitLab URL: `https://gitlab.example.com`
- 🔑 Token: Grab from GitLab Admin UI
- ⚙️ Executor: `docker`
- 🐳 Image: `alpine:latest`

You’ll be prompted interactively:

| Prompt               | Example                                |
|----------------------|----------------------------------------|
| GitLab instance URL  | https://gitlab.example.com             |
| Registration token   |  Get this from Admin > CI/CD > Runners |
| Description          | docker-remote-runner                   |
| Tags                 | docker,remote                          |
| Executor             | docker                                 |
| Default image        | alpine:latest                          |

💡 After registration, the runner will automatically connect to your GitLab instance and start handling CI jobs.

## 🔐 HTTPS & Domain Setup

- 🌐 External URL: `https://gitlab.example.com`
- 🔁 Auto-renew via Let's Encrypt
- 🚦 HTTP redirected to HTTPS
- 🛠️ SSH exposed on port `2224`

## 📦 Notes

- ⚙️ GitLab config is stored in `./config`
- ✅ All CI/CD features included
- 🔐 HTTPS can be added later
- 🏃 GitLab Runner can be added via Docker

---
