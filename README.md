# CI/CD Project

## Project Overview

This project demonstrates a **CI/CD pipeline** implementation using **Jenkins**, **Prometheus**, and **Grafana** to automate the build, test, and deployment of a React and Python application. The pipeline integrates monitoring and visualization to ensure efficient deployment and system reliability.

### Features:
- **CI/CD**: Automates the Continuous Integration (CI) and Continuous Deployment (CD) processes using **Jenkins**.
- **Monitoring**: Monitors application and container metrics using **Prometheus**.
- **Visualization**: Visualizes monitoring data for improved insights using **Grafana**.

---

## Tech Stack

- **Jenkins**: CI/CD automation tool.
- **Docker**: Containerization platform.
- **Prometheus**: Pipeline metrics tool.
- **Grafana**: Monitoring visualization tool.
- **AWS EC2**: Cloud instance for deployment.
- **React**: Frontend application.

---

## Setup Instructions

### 1. Local Setup

#### Prerequisites:
- **Docker**: Installed on your local machine.
- **Git**: Installed to clone repositories.
- **Jenkins**, **Prometheus**, **Grafana**: Installed in Docker.

#### Steps:

1. **Create a Docker Network**:
   ```bash
   docker network create jenkins
   ```

2. **Run Jenkins in Docker**:
   ```bash
   docker run --name jenkins-docker --detach --privileged --network jenkins \
     --env DOCKER_TLS_CERTDIR=/certs \
     --volume jenkins-docker-certs:/certs/client \
     --volume jenkins-data:/var/jenkins_home \
     --publish 2376:2376 docker:dind --storage-driver overlay2
   ```

3. **Create a Dockerfile for Jenkins Blue Ocean**:
   ```dockerfile
   FROM jenkins/jenkins:2.426.2-jdk17
   USER root
   RUN apt-get update && apt-get install -y lsb-release
   RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
       https://download.docker.com/linux/debian/gpg
   RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
       https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
   RUN apt-get update && apt-get install -y docker-ce-cli
   USER jenkins
   RUN jenkins-plugin-cli --plugins "blueocean:1.27.9 docker-workflow:572.v950f58993843"
   ```

4. **Run Jenkins Blue Ocean**:
   ```bash
   docker run --name jenkins-blueocean --detach --network jenkins \
     --env DOCKER_HOST=tcp://docker:2376 --env DOCKER_CERT_PATH=/certs/client \
     --env DOCKER_TLS_VERIFY=1 \
     --publish 49000:8080 --publish 50000:50000 \
     --volume jenkins-data:/var/jenkins_home \
     --volume jenkins-docker-certs:/certs/client:ro \
     myjenkins-blueocean:2.426.2-1
   ```

5. **Retrieve Administrator Password for Jenkins**:
   ```bash
   docker logs jenkins-blueocean
   ```
   ![image](https://github.com/user-attachments/assets/733c6a26-6f15-44be-a564-c88777986afe)


6. **Run Prometheus**:
   ```bash
   docker run -d --name prometheus -p 9091:9090 prom/prometheus
   ```

7. **Configure Prometheus**:
   - Open Prometheus Docker Terminal:
     ```bash
     docker exec -it prometheus sh
     ```
   - Edit `prometheus.yml`:
     ```bash
     vi /etc/prometheus/prometheus.yml
     ```
   - Add the following job at the end:
     ```yaml
     - job_name: "jenkins"
       metrics_path: /prometheus/
       static_configs:
         - targets: ["host.docker.internal:49000"]
     ```
   - Restart Prometheus:
     ```bash
     docker restart prometheus
     ```

8. **Run Grafana**:
   ```bash
   docker run -d --name grafana -p 3031:3031 -e "GF_SERVER_HTTP_PORT=3031" grafana/grafana
   ```

---

### 2. EC2 Setup

#### Prerequisites:
- **AWS EC2** instance (suggested: `t2.medium` with 8GB storage).
- **Docker** installed on the EC2 instance.
- **SSH Access**: Ensure you have the private key for SSH.

#### Steps:

1. **Create a Key Pair**:
   - Generate an RSA key pair in `.pem` format and save it locally.

2. **Configure Security Groups**:
   Add the following inbound rules:
   - **HTTPS**: Port 443, Source: Anywhere (0.0.0.0/0).
   - **HTTP**: Port 80, Source: Anywhere (0.0.0.0/0).
   - **SSH**: Port 22, Source: Anywhere (0.0.0.0/0).
   - Custom TCP:
     - Port **9000** (Nginx)
     - Port **9091** (Prometheus)
     - Port **3000** (React App)
     - Port **3031** (Grafana)
     - Port **49000** (Jenkins)
     - Port **2376**, **50000** (Jenkins internal services).

3. **Launch EC2 Instance**:
   - Use the key pair created earlier and attach the security group.

4. **Connect to EC2 via SSH**:
   ```bash
   ssh -i "your-key.pem" ec2-user@<public-ip-ec2>
   ```

5. **Install Docker on EC2**:
   ```bash
   sudo yum update -y
   sudo yum install docker -y
   sudo service docker start
   sudo usermod -a -G docker ec2-user
   ```

6. **Run Jenkins, Prometheus, and Grafana**:
   Follow the **Local Setup** instructions, prefacing commands with `sudo` as necessary.

---

## Screenshots

### Jenkins Dashboard
![jenkins dashboard](https://github.com/user-attachments/assets/8c4170e4-459a-415a-8a20-fbd2da364d79)


### Pipeline Details
![detail pipeline react-app](https://github.com/user-attachments/assets/b6f3b154-8bac-405c-94d0-f731d769fbaa)


### Grafana Dashboard
![Grafana dashboard](https://github.com/user-attachments/assets/737b41aa-3ecc-47ec-9830-d34880bc5676)


---

## Public Addresses

| Component    | URL                                   |
|--------------|---------------------------------------|
| **Jenkins**  | `http://<public-ip>:49000`           |
| **Prometheus**| `http://<public-ip>:9091`           |
| **Grafana**  | `http://<public-ip>:3031`           |
| **React App**| `http://<public-ip>:3000`           |

---
