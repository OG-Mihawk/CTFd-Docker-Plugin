# CTFd Docker Containers Plugin

<div align="center">
  <h3 align="center">CTFd Docker Containers Plugin</h3>
  <p align="center">
    A plugin to create containerized challenges for your CTF contest.
  </p>
</div>

## Table of Contents
1. [Getting Started](#getting-started)
   - [Prerequisites](#prerequisites)
   - [Installation](#installation)
2. [Usage](#usage)
   - [Using Local Docker Daemon](#using-local-docker-daemon)
   - [Using Remote Docker via SSH](#using-remote-docker-via-ssh)
3. [Creating Challenges](#creating-challenges)
   - [SSH Challenges](#ssh-challenges)
4. [Demo](#demo)
5. [Roadmap](#roadmap)
6. [License](#license)
7. [Contact](#contact)

---

> [!WARNING]
> The current cheating-detection algorithm is very slow, so it is NOT recommended for competitions with many participants.

## Getting Started

This section provides instructions for setting up the project locally.

### Prerequisites

To use this plugin, you should have:
- Experience hosting CTFd with Docker
- Basic knowledge of Docker
- SSH access to remote servers (if using remote Docker)

### Installation

1. **Clone this repository:**
   ```bash
   git clone https://github.com/phannhat17/CTFd-Docker-Plugin.git
   ```
2. **Rename the folder:**
   ```bash
   mv CTFd-Docker-Plugin containers
   ```
3. **Move the folder to the CTFd plugins directory:**
   ```bash
   mv containers /path/to/CTFd/plugins/
   ```

[Back to top](#ctfd-docker-containers-plugin)

---

## Usage

### Using Local Docker Daemon

#### Case A: **CTFd Running Directly on Host:**
  - Go to the plugin settings page: `/containers/settings`
  - Fill in all fields except the `Base URL`.

  ![Settings Example](./image-readme/1.png)

#### Case B: **CTFd Running via Docker:**
  - Map the Docker socket into the CTFd container by modify the `docker-compose.yml` file:
  ```bash
  services:
    ctfd:
      ...
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
      ...
  ```
  - Restart CTFd
  - Go to the plugin settings page: `/containers/settings`
  - Fill in all fields except the `Base URL`.

### Using Remote Docker via SSH

For remote Docker, the CTFd host must have SSH access to the remote server.

#### Prerequisites:
- **SSH access** from the CTFd host to the Docker server
- The remote server's fingerprint should be in the `known_hosts` file
- SSH key files (`id_rsa`) and an SSH config file should be available

#### Case A: **CTFd Running via Docker**

1. **Prepare SSH Config:**
   ```bash
   mkdir ssh_config
   cp ~/.ssh/id_rsa ~/.ssh/known_hosts ~/.ssh/config ssh_config/
   ```

2. **Mount SSH Config into the CTFd container:**
   ```yaml
   services:
     ctfd:
       ...
       volumes:
         - ./ssh_config:/root/.ssh:ro
       ...
   ```

3. **Restart CTFd:**
   ```bash
   docker-compose down
   docker-compose up -d
   ```

#### Case B: **CTFd Running Directly on Host**

1. **Ensure SSH Access:**
   - Test the connection:
     ```bash
     ssh user@remote-server
     ```

2. **Configure Docker Base URL:**
   - In the CTFd plugin settings page (`/containers/settings`), set:
     ```
     Base URL: ssh://user@remote-server
     ```

3. **Restart CTFd:**
   ```bash
   sudo systemctl restart ctfd
   ```

[Back to top](#ctfd-docker-containers-plugin)

---

## Creating Challenges

### SSH Challenges

SSH challenges allow participants to connect to containerized environments via SSH. Here's how to set them up:

#### 1. **Prepare Your Docker Image**

Your Docker image needs to have an SSH server configured. Here's an example Dockerfile:

```dockerfile
FROM ubuntu:22.04

# Install OpenSSH server
RUN apt-get update && \
    apt-get install -y openssh-server && \
    rm -rf /var/lib/apt/lists/*

# Create SSH directory
RUN mkdir /var/run/sshd

# Set up a user (you can customize this)
RUN useradd -m -s /bin/bash ctfuser && \
    echo 'ctfuser:ctfpassword' | chpasswd

# Optional: Allow root login or configure SSH settings
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# SSH login fix (otherwise user is kicked off after login)
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

# Expose SSH port
EXPOSE 22

# Start SSH service
CMD ["/usr/sbin/sshd", "-D"]
```

#### 2. **Build and Push Your Image**

```bash
docker build -t your-ssh-challenge:latest .
docker tag your-ssh-challenge:latest your-registry/your-ssh-challenge:latest
docker push your-registry/your-ssh-challenge:latest
```

Or ensure the image is available on the Docker host where challenges will run.

#### 3. **Create Challenge in CTFd**

1. Go to Admin Panel → Challenges → Create Challenge
2. Select "Container" as the challenge type
3. Fill in the challenge details:
   - **Name**: Your challenge name
   - **Image**: Select your SSH-enabled Docker image
   - **Connection Type**: Select "SSH"
   - **Port**: 22 (the SSH port inside the container)
   - **Initial Value**, **Decay**, **Minimum**: Set point values
   - **Flag Mode**: Choose "Static" or "Random"
     - For SSH challenges, you can inject the flag via environment variable `$FLAG`
     - Example: Place flag in a file during container startup

4. In your challenge description, provide:
   - SSH credentials (username/password)
   - Instructions on what to find
   - Any necessary hints

#### 4. **Example: Flag Injection**

Modify your Dockerfile to use the FLAG environment variable:

```dockerfile
# Add this to your Dockerfile
RUN echo '#!/bin/bash\necho "Flag: $FLAG" > /home/ctfuser/flag.txt' > /entrypoint.sh && \
    chmod +x /entrypoint.sh

# Change CMD to use entrypoint
CMD ["/bin/bash", "-c", "/entrypoint.sh && /usr/sbin/sshd -D"]
```

#### 5. **Security Considerations**

- Use non-root users when possible
- Consider using key-based authentication for more realistic scenarios
- Set resource limits (memory, CPU) in the plugin settings
- Remember that participants will have shell access - ensure proper isolation

#### 6. **Example SSH Challenge**

A complete working example is available in the `examples/ssh-challenge/` directory. This example demonstrates:
- Basic SSH server setup
- Flag injection via environment variable
- User authentication configuration

To use the example:
```bash
cd examples/ssh-challenge
docker build -t ssh-challenge:latest .
```

See the [SSH Challenge Example README](./examples/ssh-challenge/README.md) for more details.

[Back to top](#ctfd-docker-containers-plugin)

---

## Demo

### Admin Dashboard
- Manage running containers
- Filter by challenge or player

![Manage Containers](./image-readme/manage.png)

### Challenge View

**Web Access** | **TCP Access** | **SSH Access**
:-------------:|:--------------:|:-------------:
![Web](./image-readme/http.png) | ![TCP](./image-readme/tcp.png) | SSH connection via terminal

**Connection Types:**
- **Web**: HTTP-based challenges accessible through a browser
- **TCP**: Raw TCP connections using netcat or similar tools
- **SSH**: Secure shell access to containerized environments

[Back to top](#ctfd-docker-containers-plugin)

![Live Demo](./image-readme/demo.gif)

[Back to top](#ctfd-docker-containers-plugin)

---

## Roadmap

- [x] Support for user mode
- [x] Admin dashboard with team/user filtering
- [x] Compatibility with the core-beta theme
- [x] Monitor share flag 
- [ ] Monitor detail on share flag 
- [ ] Prevent container creation on solved challenge

For more features and known issues, check the [open issues](https://github.com/phannhat17/CTFd-Docker-Plugin/issues).

[Back to top](#ctfd-docker-containers-plugin)

---

## License

Distributed under the MIT License. See `LICENSE.txt` for details.

> This plugin is an upgrade of [andyjsmith's plugin](https://github.com/andyjsmith/CTFd-Docker-Plugin) with additional features.

If there are licensing concerns, please reach out via email (contact below).

[Back to top](#ctfd-docker-containers-plugin)

---

## Contact

**Phan Nhat**  
- **Discord:** ftpotato  
- **Email:** contact@phannhat.id.vn  
- **Project Link:** [CTFd Docker Plugin](https://github.com/phannhat17/CTFd-Docker-Plugin)

[Back to top](#ctfd-docker-containers-plugin)

