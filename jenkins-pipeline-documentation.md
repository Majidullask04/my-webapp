# Jenkins CI/CD Pipeline Documentation — `my-webapp`

## Overview

This document describes a Jenkins declarative pipeline that implements a DevSecOps workflow for the `my-webapp` repository. The pipeline pulls source code, runs security and quality scans, builds a Docker image, scans the image, pushes it to a registry, and deploys the application to a k3s Kubernetes cluster.

**Repository:** `https://github.com/Majidullask04/my-webapp.git`
**Docker image:** `majid04/my-webapp`
**Target environment:** k3s (lightweight Kubernetes)

---

## Pipeline Architecture

```
Git Checkout → Repo Discovery → Build → Secret Scan (Gitleaks) →
SonarQube Scan → Quality Gate → Filesystem Vuln Scan (Trivy) →
Docker Build → Image Vuln Scan (Trivy) → Docker Push → Deploy to k3s
```

Each stage acts as a gate — the pipeline is structured so security and quality checks run *before* the artifact is built and pushed, following shift-left security practice.

---

## Prerequisites

Before running this pipeline, the following must be configured on the Jenkins controller/agent:

| Requirement | Purpose |
|---|---|
| **NodeJS tool** (`nodejs26`) | Configured under Manage Jenkins → Tools |
| **SonarQube Scanner tool** (`sonar-scanner`) | Configured under Manage Jenkins → Tools |
| **SonarQube server config** (`sonar`) | Configured under Manage Jenkins → System → SonarQube servers |
| **Gitleaks** | Installed on the agent, available on `PATH` |
| **Trivy** | Installed on the agent, available on `PATH` |
| **Docker** | Installed on the agent; Jenkins user must have permission to run Docker |
| **kubectl** | Installed on the agent, with a working kubeconfig pointing to the k3s cluster |
| **Python3 + pip3** | Installed on the agent (used in the build stage) |
| **Jenkins credential: `docker-id`** | Docker registry credentials (Docker Hub) |

---

## Environment Variables

```groovy
SCANNER_HOME = tool 'sonar-scanner'      // Resolves SonarQube scanner install path
DOCKER_IMAGE = 'majid04/my-webapp'       // Docker Hub repository name
```

---

## Stage-by-Stage Breakdown

### 1. Git Checkout
Clones the `main` branch of `my-webapp` into the Jenkins workspace.

### 2. Discover Repository
Runs a set of `find` commands to detect what kind of project is present (Node, Java/Maven, Java/Gradle, Go, .NET, Python, or containerized via Dockerfile/docker-compose). This is a diagnostic step — output is informational and doesn't gate the pipeline.

### 3. Build
Installs Python dependencies via `pip3 install -r requirements.txt --break-system-packages`. The `--break-system-packages` flag is required on newer Debian/Ubuntu agents (PEP 668) where pip refuses to install into the system Python environment by default.

### 4. Gitleaks
Scans the repository for committed secrets (API keys, tokens, credentials). Runs with `|| true`, so it currently **does not fail the build** — findings are logged but not enforced.

### 5. SonarQube Scanner
Runs static code analysis and uploads results to the configured SonarQube server under project key `my-webapp`.

### 6. Quality Gate
Waits (up to 1 hour) for SonarQube's quality gate result. Currently set to `abortPipeline: false`, meaning a failed quality gate is logged but does not stop the pipeline.

### 7. Trivy Filesystem Scan
Scans the repository filesystem for known vulnerabilities (`HIGH`/`CRITICAL` severity only) in dependencies.

### 8. Docker Build
Builds the Docker image, tagged with the Jenkins `BUILD_NUMBER` for traceability: `majid04/my-webapp:<build-number>`.

### 9. Trivy Image Scan
Scans the built Docker image for vulnerabilities and writes a table-format report to `trivy-image-report.txt`.

### 10. Docker Push
Pushes the tagged image to Docker Hub using the `docker-id` Jenkins credential.

### 11. Deploy to k3s
Applies all Kubernetes manifests in the `k8s/` directory to the cluster via `kubectl apply -f k8s/`.

---

## Known Gaps / Next Improvements

- **OWASP Dependency-Check** was removed from this version of the pipeline due to long first-run download times (NVD database sync). Can be re-added with an NVD API key (`--nvdApiKey`) injected via `withCredentials` using single-quoted shell strings to avoid Groovy interpolation warnings.
- **Gitleaks and SonarQube quality gate do not currently fail the build** on findings — consider enforcing (`abortPipeline: true`, removing `|| true`) once baseline findings are triaged.
- **No image tag other than `BUILD_NUMBER`** — consider also tagging `latest` or a Git SHA for easier rollback tracking.
- **No rollback stage** — a failed deploy currently has no automated rollback (`kubectl rollout undo`).
- **kubeconfig handling** — deployment currently assumes a static kubeconfig is available to the Jenkins agent user; consider using the Kubernetes CLI plugin (`withKubeConfig`) for credential-managed access instead.

---

## Full Pipeline Source

```groovy
pipeline {
    agent any

    tools {
        nodejs 'nodejs26'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'majid04/my-webapp'
    }

    stages {
        stage('1.git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Majidullask04/my-webapp.git'
            }
        }

        stage('2.Discover Repository') {
            steps {
                sh '''
                find . -name "package.json"
                find . -name "pom.xml"
                find . -name "build.gradle"
                find . -name "go.mod"
                find . -name "*.csproj"
                find . -name "requirements.txt"
                find . -name "Dockerfile"
                find . -name "docker-compose.yml"
                '''
            }
        }

        stage('3.build') {
            steps {
                sh 'pip3 install -r requirements.txt --break-system-packages'
            }
        }

        stage('4.gitleaks') {
            steps {
                sh 'gitleaks detect source . || true '
            }
        }

        stage('5.sonar scanner') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=my-webapp\
                    -Dsonar.projectKey=my-webapp
                    '''
                }
            }
        }

        stage('6.quality gate ') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('7.Trivy FS Scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL .'
            }
        }

        stage('8.Docker build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} ."
            }
        }

        stage('9. Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.txt ${DOCKER_IMAGE}:${BUILD_NUMBER}"
            }
        }

        stage('10. Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-id') {
                        sh "docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('11. Deploy to k3s') {
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }
}
```
