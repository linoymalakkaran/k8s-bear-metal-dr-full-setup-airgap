# Jenkins CI/CD Pipeline for Kubernetes Deployments

This directory contains Jenkins pipeline configurations for building, testing, and deploying applications to the Kubernetes cluster.

## Pipeline Types

### 1. Application Build and Deploy Pipeline
- Build Docker image
- Run tests
- Push to registry
- Deploy to Kubernetes
- Run smoke tests

### 2. Helm Chart Deployment Pipeline
- Lint Helm charts
- Deploy to staging
- Run integration tests
- Deploy to production

### 3. Infrastructure Pipeline
- Validate infrastructure changes
- Apply Terraform/Ansible
- Verify cluster health

## Setup

### Prerequisites
```bash
# Install Jenkins plugins
- Kubernetes Plugin
- Docker Pipeline Plugin
- Git Plugin
- Pipeline Plugin
- Credentials Binding Plugin
```

### Configure Jenkins

1. **Add Kubernetes credentials**:
   - Go to Jenkins → Credentials → System → Global credentials
   - Add kubeconfig file
   - Add Docker registry credentials

2. **Configure Kubernetes plugin**:
   - Go to Manage Jenkins → Configure System
   - Add Kubernetes cloud configuration
   - Set kubeconfig path or cluster details

3. **Create pipeline jobs**:
   - New Item → Pipeline
   - Point to Jenkinsfile in repository
   - Configure webhooks for automatic triggers

## Usage

### Create New Pipeline

```groovy
// Jenkinsfile example
@Library('shared-pipeline-library') _

pipeline {
    agent {
        kubernetes {
            label 'build-pod'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:latest
    command: ['cat']
    tty: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['cat']
    tty: true
"""
        }
    }
    
    stages {
        stage('Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:${BUILD_NUMBER} .'
                }
            }
        }
        
        stage('Deploy') {
            steps {
                container('kubectl') {
                    sh 'kubectl apply -f k8s/'
                }
            }
        }
    }
}
```

### Run Pipeline

```bash
# Trigger manually
# Jenkins UI → Job → Build Now

# Trigger via webhook
curl -X POST http://jenkins.example.com/job/my-app/build

# Trigger via Git commit
git commit -m "Update app" && git push
```

## Environment Variables

```groovy
environment {
    DOCKER_REGISTRY = 'registry.k8s.internal'
    KUBE_NAMESPACE = 'production'
    HELM_CHART_PATH = './helm-charts/sample-app'
}
```

## Secrets Management

```groovy
// Use Jenkins credentials
withCredentials([
    usernamePassword(
        credentialsId: 'docker-registry',
        usernameVariable: 'REGISTRY_USER',
        passwordVariable: 'REGISTRY_PASS'
    ),
    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')
]) {
    // Your build steps
}
```

## Best Practices

1. **Use Kubernetes agents** for scalable builds
2. **Implement proper testing** at each stage
3. **Use semantic versioning** for images
4. **Implement rollback mechanisms**
5. **Add approval gates** for production
6. **Monitor pipeline metrics**
7. **Keep pipelines as code** in version control

## Troubleshooting

```groovy
// Debug pipeline
sh 'printenv | sort'
sh 'kubectl cluster-info'
sh 'docker images'

// Save artifacts for debugging
archiveArtifacts artifacts: 'build/**/*.log', allowEmptyArchive: true
```

## Additional Pipelines

- [GitLab CI](../gitlab-ci/README.md)
- [GitHub Actions](../github-actions/README.md)
