# SFTP Server on Docker: A Comprehensive Step-by-Step Tutorial

This guide explains how to set up and manage **SFTP servers** using Docker. It covers everything from quick, basic setups for local testing to more advanced, secure, and customizable servers for production environments using Docker Compose.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [**Part 1: Basic Setups with a Pre-Built Image**](#part-1-basic-setups-with-a-pre-built-image)
    - [A) Password-Based SFTP Server](#a-password-based-sftp-server)
    - [B) SSH Key-Based SFTP Server](#b-ssh-key-based-sftp-server)
    - [C) Running Multiple Servers or Users](#c-running-multiple-servers-or-users)
3. [**Part 2: Building a Custom SFTP Server for Production**](#part-2-building-a-custom-sftp-server-for-production)
    - [**Scenario A: Secure Key-Only SFTP Server**](#scenario-a-secure-key-only-sftp-server)
        - [1. Project Structure](#1-project-structure-key-only)
        - [2. Dockerfile (Key-Only)](#2-dockerfile-key-only)
        - [3. Docker Compose (Key-Only)](#3-docker-compose-key-only)
        - [4. Deployment Steps](#4-deployment-steps-key-only)
    - [**Scenario B: Hybrid SFTP Server (Key + Password)**](#scenario-b-hybrid-sftp-server-key--password)
        - [1. Project Structure](#1-project-structure-hybrid)
        - [2. Dockerfile (Hybrid)](#2-dockerfile-hybrid)
        - [3. Docker Compose (Hybrid)](#3-docker-compose-hybrid)
        - [4. Deployment Steps](#4-deployment-steps-hybrid)
4. [SFTP Command-Line Reference](#sftp-command-line-reference)
5. [Connecting via FileZilla](#connecting-via-filezilla)


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

## Part 1: Basic Setups with a Pre-Built Image

This method is quick for local testing using the pre-built `atmoz/sftp` image.

1.  Pull the Docker image:
    ```bash
    docker pull atmoz/sftp
    ```

### A) Password-Based SFTP Server

2.  Run a password-based SFTP container:
    ```bash
    docker run -p 2222:22 -d atmoz/sftp testuser1:password123:::upload
    ```
    - `-p 2222:22` maps host port 2222 to container SSH port 22.
    - `testuser1:password123:::upload` creates `testuser1` with `password123` and a home directory containing an `upload` folder.

3.  Connect via terminal:
    ```bash
    sftp -P 2222 testuser1@localhost
    ```

### B) SSH Key-Based SFTP Server

This also uses the pre-built image for quick local testing with SSH keys.

1.  **Generate SSH Key Pair (if needed):**
    ```bash
    ssh-keygen -t rsa -b 4096 -f ~/.ssh/sftp_test_key
    ```

2.  **Prepare Folder for Authorized Keys:**
    ```bash
    # Create the directory structure
    mkdir -p ~/sftp_key_auth/testuser2
    # Copy your public key
    cp ~/.ssh/sftp_test_key.pub ~/sftp_key_auth/testuser2/authorized_keys
    ```

3.  **Run SFTP Container with Key Auth:**
    ```bash
    docker run --platform linux/amd64 -p 2225:22 -d \
      -v ~/sftp_key_auth/testuser2/authorized_keys:/home/testuser2/.ssh/authorized_keys:ro \
      atmoz/sftp testuser2::::
    ```
    - `testuser2::::` creates user `testuser2` without a password for key-only login.
    - The `-v` flag mounts your public key, granting access.

4.  **Connect via Terminal:**
    ```bash
    sftp -i ~/.ssh/sftp_test_key -P 2225 testuser2@localhost
    ```

### C) Running Multiple Servers or Users

-   **Multiple Containers (Different Ports):**
    ```bash
    docker run -p 2223:22 -d atmoz/sftp userA:passA:::upload
    docker run -p 2224:22 -d atmoz/sftp userB:passB:::upload
    ```

-   **Multiple Users in One Container:**
    ```bash
    docker run -p 2222:22 -d atmoz/sftp \
      user1:pass1:::dir1 \
      user2:pass2:::dir2
    ```

---

## Part 2: Building a Custom SFTP Server for Production

This section details how to build a custom, secure SFTP server using your own `Dockerfile` and `docker-compose.yml`. This provides much more control and is the recommended approach for a real server.

### Scenario A: Secure Key-Only SFTP Server

This is the most secure and recommended setup. It disables passwords entirely and relies only on SSH keys.

#### 1. Project Structure (Key-Only)
Create a project directory with the following files:
```
/sftp_key_only/
├── docker-compose.yml
├── Dockerfile
└── authorized_keys
```

#### 2. Dockerfile (Key-Only)
This file builds a custom Ubuntu image with an SSH server configured for key-only authentication.

**File: `Dockerfile`**
```dockerfile
FROM ubuntu:22.04

# Install SSH server and create sshd directory
RUN apt-get update && \
    apt-get install -y openssh-server && \
    mkdir /var/run/sshd

# Create a user for SFTP access
ARG USERNAME=sftpuser
RUN useradd -m -s /bin/bash $USERNAME

# Disable password authentication for security
RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Configure SFTP by appending settings to sshd_config
RUN echo "Match User $USERNAME" >> /etc/ssh/sshd_config && \
    echo "    ForceCommand internal-sftp" >> /etc/ssh/sshd_config && \
    echo "    PasswordAuthentication no" >> /etc/ssh/sshd_config && \
    echo "    ChrootDirectory /home/$USERNAME/data" >> /etc/ssh/sshd_config && \
    echo "    AllowTcpForwarding no" >> /etc/ssh/sshd_config && \
    echo "    X11Forwarding no" >> /etc/ssh/sshd_config

# Create .ssh directory for key-based authentication
RUN mkdir -p /home/$USERNAME/.ssh && \
    chown -R $USERNAME:$USERNAME /home/$USERNAME/.ssh && \
    chmod 700 /home/$USERNAME/.ssh

# Expose SSH port and start the server
EXPOSE 22
CMD ["/usr/sbin/sshd","-D"]
```

#### 3. Docker Compose (Key-Only)
This file runs your server, mapping ports and mounting volumes for keys and data.

**File: `docker-compose.yml`**
```yaml
version: '3.0'

services:
  sftp:
    build:
      context: .
      args:
        USERNAME: sftpuser
    container_name: my-sftp-key-only-server
    ports:
      # Map host port 2223 to the container's SSH port 22
      - "2223:22"
    volumes:
      # Mount the authorized_keys file from the host (read-only)
      - ./authorized_keys:/home/sftpuser/.ssh/authorized_keys:ro
      # Mount a directory for user data
      - ./sftp_data:/home/sftpuser/data
```

#### 4. Deployment Steps (Key-Only)

1.  **Generate a Dedicated SSH Key (Local Machine):**
    ```bash
    ssh-keygen -t rsa -b 4096 -f ~/.ssh/prod_sftp_key
    ```
2.  **Add Public Key to Server:** Copy the content of your public key (`~/.ssh/prod_sftp_key.pub`) and paste it into the `authorized_keys` file on your server.
3.  **Set Host Permissions:** The container user needs ownership of the mounted files.
    ```bash
    # Create and set ownership for the data directory
    mkdir -p sftp_data
    sudo chown 1001:1001 sftp_data # UID 1001 for the created sftpuser

    # Set ownership and strict permissions for the key file
    sudo chown 1001:1001 authorized_keys
    sudo chmod 600 authorized_keys
    ```
4.  **Open Cloud Firewall:** Allow incoming TCP traffic on port `2223`.
5.  **Build and Run:** `docker-compose up -d --build`
6.  **Connect:**
    ```bash
    sftp -i ~/.ssh/prod_sftp_key -P 2223 sftpuser@YOUR_SERVER_IP
    ```

---


### Scenario B: Hybrid SFTP Server (Key + Password)

This setup requires both a valid SSH key and a password, adding a second layer of authentication.

#### 1. Project Structure (Hybrid)
```
/sftp_hybrid/
├── docker-compose.yml
├── Dockerfile
└── authorized_keys
```

#### 2. Dockerfile (Hybrid)
This `Dockerfile` creates a user and sets a password passed in during the build process.

**File: `Dockerfile`**
```dockerfile
FROM ubuntu:22.04

# Install SSH server
RUN apt-get update && \
    apt-get install -y openssh-server && \
    mkdir /var/run/sshd

# Create user and set password from build argument
ARG USERNAME=hybriduser
ARG PASSWORD=DefaultPasswordChangeMe
RUN useradd -m -s /bin/bash $USERNAME && \
    echo "$USERNAME:$PASSWORD" | chpasswd

# --- SSH Configuration ---
# Enable both public key and password authentication
RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
RUN echo "AuthenticationMethods publickey,password" >> /etc/ssh/sshd_config

# Create .ssh directory for key mounting
RUN mkdir -p /home/$USERNAME/.ssh && \
    chown -R $USERNAME:$USERNAME /home/$USERNAME/.ssh && \
    chmod 700 /home/$USERNAME/.ssh

# Expose SSH port and start the server
EXPOSE 22
CMD ["/usr/sbin/sshd","-D"]
```

#### 3. Docker Compose (Hybrid)
The `docker-compose.yml` file passes the username and password as build arguments.

**File: `docker-compose.yml`**
```yaml
version: '3.0'

services:
  sftp:
    build:
      context: .
      args:
        USERNAME: hybriduser
        PASSWORD: YourSuperStrongPassword123!
    container_name: my-sftp-hybrid-server
    ports:
      - "2222:22"
    volumes:
      - ./authorized_keys:/home/hybriduser/.ssh/authorized_keys:ro
      - ./sftp_data:/home/hybriduser/data
```

#### 4. Deployment Steps (Hybrid)

1.  **Generate SSH Key:** Ensure you have an SSH key pair.
2.  **Add Public Key:** Place the client's public key in the `authorized_keys` file.
3.  **Set Password:** **Crucially, change the default `PASSWORD`** in your `docker-compose.yml` file to a strong, unique password.
4.  **Set Permissions & Firewall:** Follow the same steps as in the key-only setup to set permissions for `sftp_data` and `authorized_keys`, and open port `2222` in your firewall.
5.  **Build and Run:** `docker-compose up -d --build`
6.  **Connect:**
    ```bash
    # Replace the path to your private key and your server IP
    sftp -i ~/.ssh/your_key -P 2222 hybriduser@YOUR_SERVER_IP
    ```
    You will be prompted for your SSH key passphrase (if any) and then for the user's password.

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
4. Port: The port you mapped (e.g., `2222`, `2223`).
5. Logon Type:
    - For **Key-Only**: `Key file`.
    - For **Hybrid**: `Interactive` (FileZilla will ask for both key and password).
6. User: The username you configured (`sftpuser`, `hybriduser`, etc.).
7. Key file: Browse and select your **private** key file.
8. Connect
