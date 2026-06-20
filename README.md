<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:00C9FF,100:92FE9D&height=220&section=header&text=Docker%20Web%20App&fontSize=58&fontColor=ffffff&fontAlignY=35&desc=A%20DevSecOps%20Pipeline%20That%20Blocks%20Vulnerable%20Code%20Before%20It%20Ships&descSize=18&descAlignY=55&animation=fadeIn" width="100%"/>

<br/>

<a href="#-architecture">Architecture</a> •
<a href="#-pipeline-flow">Pipeline</a> •
<a href="#-security-gates">Security</a> •
<a href="#-infrastructure">Infra</a> •
<a href="#-quick-start">Quick Start</a> •
<a href="#-contact">Contact</a>

<br/>

![Build](https://img.shields.io/badge/build-passing-success?style=for-the-badge&logo=jenkins&logoColor=white)
![Quality Gate](https://img.shields.io/badge/sonarqube-passed-brightgreen?style=for-the-badge&logo=sonarqube&logoColor=white)
![Security](https://img.shields.io/badge/trivy-0%20critical-success?style=for-the-badge&logo=aqua&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue?style=for-the-badge)

![GitHub last commit](https://img.shields.io/github/last-commit/jyothisai0336/Project_Docker_Web_App?style=flat-square&color=00C9FF)
![GitHub repo size](https://img.shields.io/github/repo-size/jyothisai0336/Project_Docker_Web_App?style=flat-square&color=00C9FF)
![GitHub stars](https://img.shields.io/github/stars/jyothisai0336/Project_Docker_Web_App?style=flat-square&color=00C9FF)

</div>

<br/>

## 📖 Overview

**Docker Web App** is a full-stack, multi-feature web application — sign-up/authentication, user profiles, posts, and a contacts/network layer — shipped end-to-end through a fully automated **DevSecOps pipeline**. Every commit is built, statically analyzed, scanned for vulnerabilities, and deployed across a **multi-node Docker Swarm cluster on AWS EC2**, and the pipeline physically cannot push an image to DockerHub if the code doesn't pass quality and security gates first.

This isn't a "Docker run and pray" deployment. It's gated, observable, and repeatable — average full pipeline run: **~56 seconds**.

> 💡 **The core idea:** security and quality checks aren't a report you read after the fact — they're a wall the pipeline can't get through if the code fails.

<br/>

## 🏗️ Architecture

```mermaid
flowchart LR
    A[👨‍💻 Developer Push] --> B[📦 GitHub]
    B --> C[⚙️ Jenkins Pipeline]
    C --> D[🔍 SonarQube CQA]
    D --> E{Quality Gate}
    E -->|❌ Fail| X[🛑 Pipeline Halted]
    E -->|✅ Pass| F[🐳 Docker Build]
    F --> G[🛡️ Trivy Image Scan]
    G --> H{Vulnerabilities?}
    H -->|❌ Found| X
    H -->|✅ Clean| I[📤 Push to DockerHub]
    I --> J[🚀 docker stack deploy]
    J --> K[🖥️ Swarm Manager]
    K --> L[Worker Node 1]
    K --> M[Worker Node 2]

    style E fill:#00C9FF,stroke:#333,color:#fff
    style H fill:#00C9FF,stroke:#333,color:#fff
    style X fill:#1a1a1a,stroke:#00C9FF,color:#fff
    style I fill:#22c55e,stroke:#333,color:#fff
    style J fill:#22c55e,stroke:#333,color:#fff
```

<br/>

## ⚙️ Pipeline Flow

An **8-stage Jenkins Declarative Pipeline** takes every commit from raw code to a live, multi-node deployment — averaging **~56 seconds** end to end.

| # | Stage | What Happens |
|---|-------|---------------|
| 1 | **Code** | Pulls the latest commit from GitHub |
| 2 | **CQA** | Static code analysis via SonarQube |
| 3 | **Quality Gates** | 🚦 Hard stop — pipeline halts if SonarQube flags don't clear |
| 4 | **Build** | Application build |
| 5 | **Docker_Build** | Docker image build |
| 6 | **Img-scan** | 🛡️ Trivy scans the image for CVEs before it goes anywhere near prod |
| 7 | **push** | Clean, scanned image pushed to DockerHub |
| 8 | **Stack** | `docker stack deploy` rolls the release out across the Swarm |

<details>
<summary><b>🔍 Click to see the Jenkinsfile structure</b></summary>

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Code') {
            steps {
                git branch: 'main', url: 'https://github.com/jyothisai0336/Project_Docker_Web_App.git'
            }
        }

        stage('CQA - SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=docker-web-app \
                    -Dsonar.projectKey=docker-web-app
                    '''
                }
            }
        }

        stage('Quality Gates') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Build') {
            steps {
                sh 'echo "Running application build"'
            }
        }

        stage('Docker_Build') {
            steps {
                sh 'docker build -t docker-web-app:${BUILD_NUMBER} .'
            }
        }

        stage('Img-scan') {
            steps {
                sh 'trivy image docker-web-app:${BUILD_NUMBER} --severity HIGH,CRITICAL --exit-code 1'
            }
        }

        stage('push') {
            steps {
                sh 'docker push <dockerhub-user>/docker-web-app:${BUILD_NUMBER}'
            }
        }

        stage('Stack') {
            steps {
                sh 'docker stack deploy -c docker-compose.yml webapp-stack'
            }
        }
    }
}
```

</details>

<br/>

## 🛡️ Security Gates

<table>
<tr>
<td width="50%" valign="top">

### SonarQube — Quality Gate

```
✅ Status:          PASSED
🐛 New Bugs:         0
🔓 New Vulnerabilities: 0
🔥 New Security Hotspots: 0
💳 Added Tech Debt:  0
```

Static analysis runs on every commit. If quality drops below threshold, **the pipeline does not proceed.**

</td>
<td width="50%" valign="top">

### Trivy — Image Scan

```
✅ Status:           CLEAN
🛡️ Critical CVEs:    0
⚠️  High CVEs:        0
📦 Scanned Before:    Push to DockerHub
```

No image reaches DockerHub — let alone production — without clearing this scan.

</td>
</tr>
</table>

> **Shift-left in practice, not just in theory.** The vulnerability scan sits *before* the push stage, not after. A vulnerable image is architecturally incapable of reaching the registry.

<br/>

## ✨ Features

- 🔐 **User Authentication** — sign-up and login flow
- 👤 **Profiles** — user profile management
- 📝 **Posts** — create and view posts
- 🤝 **Contacts / Network** — connect with other users
- 🗄️ **Persistent storage** — relational database backing the application data

<br/>

## 🖥️ Infrastructure

<div align="center">

| Node | Role | Status |
|------|------|--------|
| 🔧 Jenkins | CI/CD Orchestrator | 🟢 Running |
| 🧠 Master | Swarm Manager | 🟢 Running |
| ⚙️ Worker 1 | Swarm Worker Node | 🟢 Running |
| ⚙️ Worker 2 | Swarm Worker Node | 🟢 Running |

</div>

All nodes run on **AWS EC2**, orchestrated as a self-managed **Docker Swarm cluster** — one manager, two workers — with Docker Compose stack files defining service placement and Swarm handling distribution across the cluster.

<br/>

## 🧰 Tech Stack

<div align="center">

![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-4E9BCD?style=for-the-badge&logo=sonarqube&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-1904DA?style=for-the-badge&logo=aquasecurity&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Docker Swarm](https://img.shields.io/badge/Docker%20Swarm-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![DockerHub](https://img.shields.io/badge/DockerHub-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/AWS%20EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Database](https://img.shields.io/badge/Database-MySQL%20%2F%20PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)

</div>

<br/>

## 🚀 Quick Start

```bash
# Clone the repository
git clone https://github.com/jyothisai0336/Project_Docker_Web_App.git
cd Project_Docker_Web_App

# Build the image locally
docker build -t docker-web-app:local .

# Run standalone
docker run -d -p 3000:3000 docker-web-app:local

# OR deploy as a Swarm stack (requires an initialized swarm)
docker stack deploy -c docker-compose.yml webapp-stack
```

<details>
<summary><b>🔧 Prerequisites</b></summary>

- Docker Engine (Swarm mode enabled for cluster deployment)
- Jenkins with SonarQube Scanner + Trivy installed on agents
- A reachable SonarQube server for the CQA stage
- A MySQL/PostgreSQL instance for persistent storage
- AWS EC2 instances (or any Docker-capable hosts) for the Swarm nodes

</details>

<br/>

## 📸 Pipeline in Action

<table>
<tr>
<td width="50%" valign="top">
<p align="center"><b>Jenkins Stage View</b></p>
<img src="screenshots/jenkins-pipeline.png" width="100%"/>
</td>
<td width="50%" valign="top">
<p align="center"><b>SonarQube Quality Gate</b></p>
<img src="screenshots/sonarqube-passed.png" width="100%"/>
</td>
</tr>
</table>

<p align="center"><i>Add your own screenshots to a <code>screenshots/</code> folder in the repo, then update the paths above — see the Quick Start section for how images get referenced in this README.</i></p>

<br/>

## 📌 Known Limitations

- This is a **reference/demo deployment** — infrastructure is provisioned for demonstration and is not kept running permanently.
- No TLS termination is configured on the demo instances; a production deployment would sit behind a load balancer or reverse proxy with HTTPS.

<br/>

## 📬 Contact

<div align="center">

**Jyothisai Mekala**
DevOps / DevSecOps Engineer

[![Email](https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:mekalajyothisai8@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/jyothisai-mekala)

<br/>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:00C9FF,100:92FE9D&height=100&section=footer" width="100%"/>

</div>
