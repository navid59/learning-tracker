# SFTP Server on Docker - Step-by-Step Tutorial

This guide explains how to quickly set up **SFTP servers** using Docker, including both **password-based** and **SSH key-based** authentication. Perfect for testing scripts or local development.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Password-Based SFTP Server](#password-based-sftp-server)
3. [SSH Key-Based SFTP Server](#ssh-key-based-sftp-server)
4. [Running Multiple SFTP Servers](#running-multiple-sftp-servers)
5. [Connecting via Terminal](#connecting-via-terminal)
6. [Connecting via FileZilla](#connecting-via-filezilla)

---

## Prerequisites
- macOS, Linux, or Windows machine with **Docker installed**
- Basic familiarity with terminal commands

Verify Docker installation:
```bash
docker --version
```

---

## Password-Based SFTP Server

1. Pull the official Docker image:
```bash
docker pull atmoz/sftp
```

2. Run a password-based SFTP container:
```bash
docker run -p 2222:22 -d atmoz/sftp testuser1:password1:::upload
```

- `-p 2222:22` maps host port 2222 to container SSH port 22
- `testuser1:password1:::upload` creates `testuser1` with password `password1` and folder `/home/testuser1/upload`

3. Verify the container:
```bash
docker ps
```

4. Connect via terminal:
```bash
sftp -P 2222 testuser1@localhost
```

---

## SSH Key-Based SFTP Server

### 1. Generate SSH Key Pair (if needed)
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/sftp_test_key
```

- This creates `sftp_test_key` (private) and `sftp_test_key.pub` (public)

### 2. Prepare Folder for Authorized Keys
```bash
mkdir -p ~/sftp_key_auth/testuser/.ssh
cp ~/.ssh/sftp_test_key.pub ~/sftp_key_auth/testuser/.ssh/authorized_keys
```

### 3. Run SFTP Container with Key Auth Only
```bash
docker run --platform linux/amd64 -p 2225:22 -d \
  -v ~/sftp_key_auth/testuser:/home/testuser \
  -v ~/sftp_key_auth/testuser/.ssh/authorized_keys:/home/testuser/.ssh/authorized_keys:ro \
  atmoz/sftp testuser::1001
```

- `testuser::1001` → creates user `testuser` without a password (key-only login)
- `-v ...` mounts the home directory and authorized keys

### 4. Verify Connection via Terminal
```bash
sftp -i ~/.ssh/sftp_test_key -P 2225 testuser@localhost
```

### 5. Fix FileZilla Key Issues (macOS)
```bash
ssh-keygen -p -m PEM -f ~/.ssh/sftp_test_key
```
- Then select this key in FileZilla for SFTP connection.

---

## Running Multiple SFTP Servers

### Option 1: Multiple Containers (Different Ports)
```bash
docker run -p 2223:22 -d atmoz/sftp testuser2:password2:::upload
```
```bash
docker run -p 2224:22 -d atmoz/sftp testuser3:password3:::upload
```

### Option 2: Multiple Users in One Container
```bash
docker run -p 2222:22 -d atmoz/sftp \
  testuser1:password1:::upload1 \
  testuser2:password2:::upload2 \
  testuser3:password3:::upload3
```
- Each user has its own folder, same port

---

## Connecting via Terminal
```bash
# Password-based
sftp -P 2222 testuser1@localhost

# Key-based
sftp -i ~/.ssh/sftp_test_key -P 2225 testuser@localhost
```

---

## Connecting via FileZilla
1. Open File → Site Manager → New Site
2. Protocol: **SFTP – SSH File Transfer Protocol**
3. Host: `localhost`
4. Port: 2222 / 2225 (depending on container)
5. User: `testuser` / `testuser1` etc.
6. Logon Type: **Key file** (or **Normal** for password-based)
7. Key file: select your private key (`sftp_test_key`) if using key auth
8. Connect

---

## Notes
- **FileZilla on macOS** requires **PEM format** for SSH keys.
- Use `chmod 600` on private keys to ensure SSH accepts them.
- Ports must be unique if running multiple containers.
- Data persistence: mount local folders using `-v /host/path:/home/user/upload`.

---

Now you have a fully functional **local SFTP environment** for testing scripts, automation, or file transfer workflows.