// Register parameters on the job immediately (so the API sees them)
properties([
  parameters([
    string(name: 'GITHUB_URL',   defaultValue: '', description: 'GitHub repo URL'),
    string(name: 'PROJECT_NAME', defaultValue: '', description: 'slug ACR/K8s'),
    string(name: 'DEPLOYMENT_ID', defaultValue: '', description: 'optional'),
    string(name: 'LB_RG', defaultValue: 'MC_devops-monitoring-rg_devops-aks_eastus', description: 'AKS managed RG (for Azure LB annotation)'),
    string(name: 'LB_IP', defaultValue: '', description: 'Optional static public IP (blank = AKS allocates)')
  ])
])

pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    durabilityHint('MAX_SURVIVABILITY') // survive controller/agent restarts
  }

  parameters {
    string(name: 'GITHUB_URL',   defaultValue: '', description: 'GitHub repository URL (https://github.com/you/repo)')
    string(name: 'PROJECT_NAME', defaultValue: '', description: 'Short project slug (used for ACR repo, K8s app name)')
    string(name: 'DEPLOYMENT_ID', defaultValue: '', description: 'Optional deployment ID for callbacks to backend')
    string(name: 'LB_RG', defaultValue: 'MC_devops-monitoring-rg_devops-aks_eastus', description: 'AKS managed rg (for Azure LB annotation)')
    string(name: 'LB_IP', defaultValue: '', description: 'Optional static public IP from Azure (empty to let AKS allocate)')
  }

  environment {
    // --- EDIT ONLY THE REGISTRY SERVER IF YOUR NAME CHANGES ---
    ACR_SERVER   = 'devopsmonitoracrrt2y5a.azurecr.io'

    PROJECT_NAME = "${params.PROJECT_NAME}"
    GITHUB_URL   = "${params.GITHUB_URL}"
    IMAGE_TAG    = "${BUILD_NUMBER}"
    NAMESPACE    = "${params.PROJECT_NAME}-dev"

    // Your backend (running on the Jenkins VM) for stage/status callbacks
    BACKEND_BASE = 'http://localhost:4000'
  }

  stages {

    stage('Validate Parameters') {
      steps {
        echo "🔍 Validating parameters…"
        script {
          if (!env.GITHUB_URL?.trim())   { error('GITHUB_URL is required') }
          if (!env.PROJECT_NAME?.trim()) { error('PROJECT_NAME is required') }

          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"validate","status":"success"}' || true
            """
          }
        }
      }
    }

    stage('Checkout') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"checkout","status":"running"}' || true
            """
          }
        }

        echo "📥 Cloning: ${env.GITHUB_URL}"
        deleteDir()

        script {
          try {
            git branch: 'main', url: env.GITHUB_URL
          } catch (err) {
            echo "main not found, trying master…"
            git branch: 'master', url: env.GITHUB_URL
          }

          env.GIT_COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.GIT_AUTHOR      = sh(script: 'git log -1 --pretty=%an',    returnStdout: true).trim()

          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"checkout","status":"success"}' || true
            """
          }
        }
      }
    }

    stage('Detect Project Type') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"analyze","status":"running"}' || true
            """
          }

          sh 'echo "📁 Files:" && ls -la'

          if (fileExists('package.json')) {
            env.PROJECT_TYPE = 'nodejs'
            env.PORT = '3000'
            echo "🔬 Detected: Node.js"
          } else if (fileExists('requirements.txt')) {
            env.PROJECT_TYPE = 'python'
            env.PORT = '8000'
            echo "🔬 Detected: Python"
          } else if (fileExists('index.html')) {
            env.PROJECT_TYPE = 'static'
            env.PORT = '80'
            echo "🔬 Detected: Static"
          } else {
            env.PROJECT_TYPE = 'static'
            env.PORT = '80'
            echo "🔬 Unknown → Static"
          }

          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"analyze","status":"success"}' || true
            """
          }
        }
      }
    }

    stage('Generate Dockerfile') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"prepare","status":"running"}' || true
            """
          }

          if (!fileExists('Dockerfile')) {
            echo "📝 Creating Dockerfile for ${env.PROJECT_TYPE}"
            def df = (
              env.PROJECT_TYPE == 'nodejs'  ? '''FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production || npm install
COPY . .
EXPOSE 3000
CMD ["npm","start"]''' :
              env.PROJECT_TYPE == 'python' ? '''FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt* ./
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python","app.py"]''' :
              /* static */                 '''FROM nginx:alpine
COPY . /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx","-g","daemon off;"]'''
            )
            writeFile file: 'Dockerfile', text: df
          } else {
            echo '📄 Using existing Dockerfile'
          }

          sh 'echo "---- Dockerfile ----"; cat Dockerfile; echo "-------------------"'

          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"prepare","status":"success"}' || true
            """
          }
        }
      }
    }

    stage('Build Image') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"build","status":"running"}' || true
            """
          }
        }

        sh """
          docker build -t ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG} .
          docker tag   ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG} ${ACR_SERVER}/${PROJECT_NAME}:latest
        """

        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"build","status":"success"}' || true
            """
          }
        }
      }
    }

    stage('Push to ACR') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"push","status":"running"}' || true
            """
          }
        }

        withCredentials([usernamePassword(credentialsId: 'acr-credentials',
                                          usernameVariable: 'ACR_USERNAME',
                                          passwordVariable: 'ACR_PASSWORD')]) {
          sh """
            echo "${ACR_PASSWORD}" | docker login "${ACR_SERVER}" -u "${ACR_USERNAME}" --password-stdin
            docker push ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG}
            docker push ${ACR_SERVER}/${PROJECT_NAME}:latest
          """
        }

        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """
              curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"push","status":"success"}' || true
            """
          }
        }
      }
    }

    stage('Setup kubectl (if needed)') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            if ! command -v kubectl >/dev/null 2>&1 && [ ! -x "./kubectl" ]; then
              echo "Installing kubectl locally..."
              VER=$(curl -fsSL https://dl.k8s.io/release/stable.txt)
              curl -fsSL -o kubectl "https://dl.k8s.io/release/${VER}/bin/linux/amd64/kubectl"
              chmod +x kubectl
            fi
            export PATH="$PWD:$PATH"
            export KUBECONFIG="${KUBECONFIG_FILE}"
            kubectl version --client
          '''
        }
      }
    }

    stage('Deploy to AKS (Rolling)') {
      steps {
        withCredentials([
          file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE'),
          usernamePassword(credentialsId: 'acr-credentials', usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD')
        ]) {
          script {
            if (params.DEPLOYMENT_ID?.trim()) {
              sh """
                curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                     -H "Content-Type: application/json" \
                     -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"deploy","status":"running"}' || true
              """
            }

            // Optional static IP line
            String lbIpLine = params.LB_IP?.trim() ? "  loadBalancerIP: ${params.LB_IP}\n" : ""

            // Compose the manifest (annotations live under metadata)
            String k8sYaml = """
apiVersion: v1
kind: Namespace
metadata:
  name: ${env.NAMESPACE}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${env.PROJECT_NAME}
  namespace: ${env.NAMESPACE}
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: ${env.PROJECT_NAME}
  template:
    metadata:
      labels:
        app: ${env.PROJECT_NAME}
    spec:
      imagePullSecrets:
      - name: acr-auth
      containers:
      - name: ${env.PROJECT_NAME}
        image: ${env.ACR_SERVER}/${env.PROJECT_NAME}:${env.IMAGE_TAG}
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: ${env.PORT}
        readinessProbe:
          httpGet:
            path: /
            port: ${env.PORT}
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /
            port: ${env.PORT}
          initialDelaySeconds: 30
          periodSeconds: 10
        resources:
          requests: { memory: "128Mi", cpu: "100m" }
          limits:   { memory: "512Mi", cpu: "500m" }
---
apiVersion: v1
kind: Service
metadata:
  name: ${env.PROJECT_NAME}-service
  namespace: ${env.NAMESPACE}
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-resource-group: ${params.LB_RG}
spec:
  selector:
    app: ${env.PROJECT_NAME}
  ports:
  - port: 80
    targetPort: ${env.PORT}
  type: LoadBalancer
${lbIpLine}""".stripIndent()

            // Apply: create ns + secret, then apply manifests with retries/timeouts
            sh '''
              set -e
              export PATH="$PWD:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"

              kubectl get ns "${NAMESPACE}" >/dev/null 2>&1 || kubectl create ns "${NAMESPACE}"

              kubectl -n "${NAMESPACE}" create secret docker-registry acr-auth \
                --docker-server="${ACR_SERVER}" \
                --docker-username="${ACR_USERNAME}" \
                --docker-password="${ACR_PASSWORD}" \
                --dry-run=client -o yaml | kubectl apply -f -
            '''

            writeFile file: 'k8s.yaml', text: k8sYaml

            timeout(time: 15, unit: 'MINUTES') {
              retry(2) {
                sh '''
                  set -e
                  export PATH="$PWD:$PATH"
                  export KUBECONFIG="${KUBECONFIG_FILE}"
                  kubectl apply -f k8s.yaml
                '''
              }
            }

            timeout(time: 10, unit: 'MINUTES') {
              retry(1) {
                sh '''
                  export PATH="$PWD:$PATH"
                  export KUBECONFIG="${KUBECONFIG_FILE}"
                  kubectl rollout status deployment/${PROJECT_NAME} -n ${NAMESPACE} --timeout=300s
                '''
              }
            }

            if (params.DEPLOYMENT_ID?.trim()) {
              sh """
                curl -s -X POST "${BACKEND_BASE}/api/internal/stages" \
                     -H "Content-Type: application/json" \
                     -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"deploy","status":"success"}' || true
              """
            }
          }
        }
      }
    }

    stage('Get External URL') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export PATH="$PWD:$PATH"
            export KUBECONFIG="${KUBECONFIG_FILE}"

            for i in {1..30}; do
              EXTERNAL_IP=$(kubectl get svc ${PROJECT_NAME}-service -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || echo "")
              if [ -n "$EXTERNAL_IP" ] && [ "$EXTERNAL_IP" != "null" ]; then
                echo "LIVE: http://$EXTERNAL_IP"
                if [ -n "${DEPLOYMENT_ID}" ]; then
                  curl -s -X POST "${BACKEND_BASE}/api/internal/deployments/${DEPLOYMENT_ID}/url" \
                       -H "Content-Type: application/json" \
                       -d "{\"external_url\":\"http://$EXTERNAL_IP\"}" || true
                fi
                break
              fi
              echo "⏳ waiting LB IP ($i/30)"
              sleep 10
            done

            kubectl get all -n ${NAMESPACE}
          '''
        }
      }
    }
  }

  post {
    always {
      sh '''
        docker rmi ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG} 2>/dev/null || true
        docker rmi ${ACR_SERVER}/${PROJECT_NAME}:latest      2>/dev/null || true
      '''
    }
    success {
      script {
        if (params.DEPLOYMENT_ID?.trim()) {
          sh """
            curl -s -X POST "${BACKEND_BASE}/api/internal/deployments/${params.DEPLOYMENT_ID}/status" \
                 -H "Content-Type: application/json" \
                 -d '{"status":"success"}' || true
          """
        }
      }
      echo "🎉 ✅ DEPLOYMENT SUCCESSFUL"
    }
    failure {
      // quick diagnostics so you can see what went wrong without SSH
      withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
        sh '''
          export PATH="$PWD:$PATH"
          export KUBECONFIG="${KUBECONFIG_FILE}"
          echo "---- Describe deployment ----"
          kubectl -n ${NAMESPACE} describe deploy/${PROJECT_NAME} || true
          echo "---- Recent events ----"
          kubectl -n ${NAMESPACE} get events --sort-by=.lastTimestamp | tail -n 50 || true
          echo "---- Pods ----"
          kubectl -n ${NAMESPACE} get pods -o wide || true
        '''
      }
      script {
        if (params.DEPLOYMENT_ID?.trim()) {
          sh """
            curl -s -X POST "${BACKEND_BASE}/api/internal/deployments/${params.DEPLOYMENT_ID}/status" \
                 -H "Content-Type: application/json" \
                 -d '{"status":"failed"}' || true
          """
        }
      }
      echo "💥 ❌ DEPLOYMENT FAILED"
    }
  }
}
