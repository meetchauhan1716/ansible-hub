# Ansible Node Exporter

## Setting Up Prometheus Node Exporter in Ansible

### Overview
This guide walks you through setting up Prometheus Node Exporter on a remote server using Ansible. The process involves creating an inventory host file, configuring Ansible, and running a playbook to install and configure Node Exporter.

### Step 1: Setting Up Ansible Inventory File
The inventory file in Ansible defines which servers are managed by Ansible. In this example, we will create an inventory file to target a specific server.

#### 1. Create an Inventory File
The default location for the Ansible inventory file is `/etc/ansible/hosts`. Open this file (or create a new one) and add your server details:

```ini
[all]
172.17.16.46 ansible_user=<your_ssh_user> ansible_ssh_private_key_file=<path_to_your_private_key>
```

- `[all]`: Group name for all the hosts.
- `172.17.16.46`: Replace this with the IP address of your remote server.
- `ansible_user=<your_ssh_user>`: Specify the user to use when connecting via SSH.
- `ansible_ssh_private_key_file=<path_to_your_private_key>`: Provide the path to your SSH private key for authentication.

**Note:** Ensure that the user you are using has SSH access to the remote server.

#### 2. Save the file and exit the editor.

---

### Step 2: Checking for SSH and Generating SSH Keys
If you don't already have an SSH key, you'll need to generate one for Ansible to connect to the remote server securely. Follow these steps:

#### 1. Check for Existing SSH Keys
Run the following command to see if you already have SSH keys:

```bash
ls ~/.ssh/id_rsa
```

- If the file exists, you can use it for your Ansible configuration.
- If not, you will need to generate a new key.

#### 2. Generate a New SSH Key (If Needed)
If you don't have an SSH key, generate one by running:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
```

- Follow the prompts (you can press Enter to accept the default file location and leave the passphrase empty).
- This will create a new pair of keys (`id_rsa` and `id_rsa.pub`) in your `~/.ssh/` directory.

#### 3. Copy the SSH Key to the Remote Server
To use the key for SSH authentication, copy your public key to the remote server:

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub <your_ssh_user>@<your_server_ip>
```

This allows Ansible to authenticate with the remote server using the private key.

---

### Step 3: Setting Up Prometheus Node Exporter Using Ansible Playbook
Now that your inventory file and SSH keys are configured, you can set up Prometheus Node Exporter on the remote server using Ansible.

#### 1. Create a Directory for Your Ansible Role
Create the folder structure for your Ansible project:

```plaintext
node_exporter/
├── playbook.yml
├── roles/
    └── node_exporter/
        ├── tasks/
        │   └── main.yml
        ├── templates/
        │   └── node_exporter.service.j2
        ├── defaults/
        │   └── main.yml
```

#### 2. Create the Playbook (`playbook.yml`)
The playbook will define the tasks to install and configure Node Exporter. Example content:

```yaml
- name: Set up Prometheus Node Exporter
  hosts: all
  become: true
  roles:
    - node_exporter
```

#### 3. Create the Role Tasks (`roles/node_exporter/tasks/main.yml`)
The role tasks will include steps like downloading the Node Exporter tarball, extracting it, moving the binary to the correct location, cleaning up, and setting up the systemd service:

```yaml
- name: Download Node Exporter tarball
  ansible.builtin.get_url:
    url: "https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz"
    dest: "/tmp/node_exporter-1.8.2.linux-amd64.tar.gz"

- name: Extract Node Exporter tarball
  ansible.builtin.unarchive:
    src: "/tmp/node_exporter-1.8.2.linux-amd64.tar.gz"
    dest: "/tmp"
    remote_src: yes

- name: Move Node Exporter binary to /usr/local/bin
  ansible.builtin.copy:
    src: "/tmp/node_exporter-1.8.2.linux-amd64/node_exporter"
    dest: "/usr/local/bin/node_exporter"
    remote_src: yes
    mode: "0755"

- name: Clean up temporary files
  ansible.builtin.file:
    path: "/tmp/node_exporter-1.8.2.linux-amd64"
    state: absent

- name: Copy systemd service file
  ansible.builtin.template:
    src: "node_exporter.service.j2"
    dest: "/etc/systemd/system/node_exporter.service"
    mode: "0644"

- name: Reload systemd daemon
  ansible.builtin.systemd:
    daemon_reload: yes

- name: Enable and start Node Exporter service
  ansible.builtin.systemd:
    name: node_exporter
    enabled: yes
    state: started
```

#### 4. Create the Systemd Service Template (`roles/node_exporter/templates/node_exporter.service.j2`)
This template will define the systemd service for Node Exporter:

```ini
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.listen-address=:9100
Restart=on-failure
RestartSec=5s
LimitNOFILE=8192

[Install]
WantedBy=multi-user.target
```

#### 5. Create Default Variables File (`roles/node_exporter/defaults/main.yml`)
Define default variables for Node Exporter settings (if needed):

```yaml
node_exporter_version: "1.8.2"
node_exporter_listen_address: ":9100"
```

---

### Step 4: Running the Playbook
Once your playbook, role, and inventory file are set up, run the playbook with the following command:

```bash
ansible-playbook playbook.yml --ask-become-pass
```

- The `--ask-become-pass` option prompts you for the sudo password to run the tasks with elevated privileges.
- Ansible will download, install, configure, and start the Node Exporter service on the remote server.

### Conclusion
After running the playbook, Node Exporter will be installed and configured on your target server. It will start as a service, allowing Prometheus to scrape system-level metrics from the remote server.

**Note:** Ensure your server's firewall allows access to port `9100` to expose the Node Exporter metrics.
