# SFTP Server on Docker - Step-by-Step Tutorial

This guide explains how to quickly set up **SFTP servers** using Docker. It covers a basic setup for local testing and a more advanced, secure setup for a production-like server using Docker Compose.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Basic: Password-Based SFTP Server](#basic-password-based-sftp-server)
3. [Basic: SSH Key-Based SFTP Server](#basic-ssh-key-based-sftp-server)
4. [Advanced: Custom SFTP Server for Production](#advanced-custom-sftp-server-for-production)
5. [SFTP Command-Line Reference](#sftp-command-line-reference)
6. [Connecting via FileZilla](#connecting-via-filezilla)
7. [Running Multiple SFTP Servers](#running-multiple-sftp-servers)

---

## Prerequisites
- A local machine or server with **Docker** and **Docker Compose** installed.
- Basic familiarity with terminal commands.

Verify Docker installation:
```bash
docker --version
docker-compose --version
```

---

## Basic: Password-Based SFTP Server

This method is quick for local testing using a pre-built image.

1. Pull the official Docker image:
```bash
docker pull atmoz/sftp
```

2. Run a password-based SFTP container:
```bash
docker run -p 2222:22 -d atmoz/sftp testuser1:password1:::upload
```
- `-p 2222:22` maps host port 2222 to container SSH port 22.
- `testuser1:password1:::upload` creates `testuser1` with `password1` and a home directory containing an `upload` folder.

3. Connect via terminal:
```bash
sftp -P 2222 testuser1@localhost
```

---

## Basic: SSH Key-Based SFTP Server

This also uses the pre-built image for quick local testing with SSH keys.

1. **Generate SSH Key Pair (if needed):**
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/sftp_test_key
```
- This creates `sftp_test_key` (private) and `sftp_test_key.pub` (public).

2. **Prepare Folder for Authorized Keys:**
```bash
mkdir -p ~/sftp_key_auth/testuser/.ssh
cp ~/.ssh/sftp_test_key.pub ~/sftp_key_auth/testuser/.ssh/authorized_keys
```

3. **Run SFTP Container with Key Auth Only:**
```bash
docker run --platform linux/amd64 -p 2225:22 -d \
  -v ~/sftp_key_auth/testuser:/home/testuser \
  -v ~/sftp_key_auth/testuser/.ssh/authorized_keys:/home/testuser/.ssh/authorized_keys:ro \
  atmoz/sftp testuser::1001
```
- `testuser::1001` creates user `testuser` without a password (key-only login).
- `-v ...` mounts the home directory and authorized keys file.

4. **Connect via Terminal:**
```bash
sftp -i ~/.ssh/sftp_test_key -P 2225 testuser@localhost
```

---

## Advanced: Custom SFTP Server for Production

This section details how to build a custom, secure, key-only SFTP server using your own `Dockerfile` and `docker-compose.yml`. This provides much more control and is the recommended approach for a real server.

### 1. Project Structure

On your server, create a project directory with the following files:

```
/your_project_directory/
├── docker-compose.yml
├── Dockerfile
├── .dockerignore
└── authorized_keys
```

### 2. The `Dockerfile`

This file builds a custom Ubuntu image with an SSH server configured for key-only authentication.

**File: `Dockerfile`**
```dockerfile
FROM ubuntu:22.04

# Install SSH server
RUN apt-get update && \
    apt-get install -y openssh-server && \
    mkdir /var/run/sshd

# Create user
ARG USERNAME=testuser
RUN useradd -m -s /bin/bash $USERNAME

# Disable password authentication for security
RUN sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# --- Key Management ---
# Create the .ssh directory and set permissions.
# The authorized_keys file will be mounted as a volume from the host.
RUN mkdir -p /home/$USERNAME/.ssh && \
    chown -R $USERNAME:$USERNAME /home/$USERNAME/.ssh && \
    chmod 700 /home/$USERNAME/.ssh

# Expose SSH port
EXPOSE 22

# Start SSH
CMD ["/usr/sbin/sshd","-D"]
```

### 3. The `docker-compose.yml` File

This file makes it easy to run and manage your server. It defines the port, volumes for keys and data, and tells Docker how to build the image.

**File: `docker-compose.yml`**
```yaml
version: '3.0'

services:
  sftp:
    build:
      context: .
    container_name: my-sftp-server
    ports:
      # Map a non-standard port (e.g., 2222) on your server to port 22 in the container
      - "2222:22"
    volumes:
      # Mount the authorized_keys file from the host into the container
      # :ro makes it read-only inside the container for extra security
      - ./authorized_keys:/home/testuser/.ssh/authorized_keys:ro
      # Mount a volume for persistent user data
      - ./sftp_data:/home/testuser/data
```

### 4. The `.dockerignore` File

This file prevents a build error by telling Docker not to access the `authorized_keys` file during the build process.

**File: `.dockerignore`**
```
authorized_keys
```

### 5. Server Setup and Deployment

Follow these steps on your production server.

1.  **Generate a Dedicated SSH Key (Local Machine):**
    ```bash
    # This creates ~/.ssh/prod_sftp_key (private) and ~/.ssh/prod_sftp_key.pub (public)
    ssh-keygen -t rsa -b 4096 -f ~/.ssh/prod_sftp_key
    ```

2.  **Copy Your Public Key:** Display the public key on your local machine and copy the output string.
    ```bash
    cat ~/.ssh/prod_sftp_key.pub
    ```

3.  **Prepare Server Files:** On your server, create the `authorized_keys` file and paste the public key into it.

4.  **Set Host Permissions:** This is critical. The user inside the container (UID 1000) needs ownership of the mounted files.
    ```bash
    # Set ownership for the key file
    sudo chown 1000:1000 authorized_keys
    # Set strict permissions for the key file
    sudo chmod 600 authorized_keys

    # Create and set ownership for the data directory
    mkdir -p sftp_data
    sudo chown 1000:1000 sftp_data
    ```

5.  **Open Cloud Firewall:** In your cloud provider (GCP, AWS, etc.), create a firewall rule to **allow incoming TCP traffic on port 2222**.

6.  **Build and Run:**
    ```bash
    docker-compose up -d --build
    ```

7.  **Connect:** From your local machine, connect using your new private key.
    ```bash
    sftp -i ~/.ssh/prod_sftp_key -P 2222 testuser@YOUR_SERVER_IP
    ```

---

## SFTP Command-Line Reference

Once you are connected and see the `sftp>` prompt, use these commands.

| Command | Description | Example |
|---|---|---|
| `put` | Upload a single file from local to remote. | `put local_file.txt` |
| `get` | Download a single file from remote to local. | `get remote_file.txt` |
| `mput` | Upload multiple files (use with wildcards). | `mput *.jpg` |
| `mget` | Download multiple files (use with wildcards). | `mget *.pdf` |
| `put -r` | Upload a directory and its contents. | `put -r local_directory` |
| `get -r` | Download a directory and its contents. | `get -r remote_directory` |
| `ls` | List files in the **remote** directory. | `ls -l` |
| `lls` | List files in the **local** directory. | `lls -l` |
| `cd` | Change the **remote** directory. | `cd /data/uploads` |
| `lcd` | Change the **local** directory. | `lcd ~/Documents` |
| `rm` | Delete a **remote** file. | `rm old_file.log` |
| `mkdir`| Create a **remote** directory. | `mkdir new_folder` |
| `pwd` | Show the current **remote** directory path. | `pwd` |
| `lpwd` | Show the current **local** directory path. | `lpwd` |
| `exit` | Disconnect from the server. | `exit` |

---

## Connecting via FileZilla
1. Open File → Site Manager → New Site
2. Protocol: **SFTP – SSH File Transfer Protocol**
3. Host: Your server's IP address or `localhost`
4. Port: `2222` (or the port you mapped)
5. User: `testuser`
6. Logon Type: **Key file**
7. Key file: Browse and select your **private** key file (e.g., `prod_sftp_key`)
8. Connect

---

## Running Multiple SFTP Servers

This applies to the basic `atmoz/sftp` image.

### Option 1: Multiple Containers (Different Ports)
```bash
docker run -p 2223:22 -d atmoz/sftp user2:pass2:::upload
docker run -p 2224:22 -d atmoz/sftp user3:pass3:::upload
```

### Option 2: Multiple Users in One Container
```bash
docker run -p 2222:22 -d atmoz/sftp \
  user1:pass1:::upload1 \
  user2:pass2:::upload2 \
  user3:pass3:::upload3
```
- Each user has its own folder, same port.