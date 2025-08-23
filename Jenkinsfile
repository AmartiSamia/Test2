// -------------------- Jenkinsfile (copy/paste) --------------------
properties([
  parameters([
    string(name: 'GITHUB_URL',   defaultValue: '', description: 'GitHub repository URL (https://github.com/you/repo)'),
    string(name: 'PROJECT_NAME', defaultValue: '', description: 'Short slug; used for ACR repo, K8s names'),
    string(name: 'DEPLOYMENT_ID', defaultValue: '', description: 'Optional backend deployment id for callbacks'),
    string(name: 'INGRESS_IP',   defaultValue: '172.191.181.1', description: 'Public IP used by ingress-nginx (x.x.x.x)')
  ])
])

pipeline {
  agent any
  options {
    disableConcurrentBuilds()
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    durabilityHint('MAX_SURVIVABILITY')   // survives Jenkins restarts
  }

  environment {
    ACR_SERVER   = 'devopsmonitoracrrt2y5a.azurecr.io'
    PROJECT_NAME = "${params.PROJECT_NAME}"
    GITHUB_URL   = "${params.GITHUB_URL}"
    IMAGE_TAG    = "${BUILD_NUMBER}"
    NAMESPACE    = "${params.PROJECT_NAME}-dev"
    BASE_DOMAIN  = "${params.INGRESS_IP}.nip.io"
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
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"checkout","status":"running"}' || true"""
          }
        }
        echo "📥 Cloning: ${env.GITHUB_URL}"
        deleteDir()
        script {
          try { git branch: 'main', url: env.GITHUB_URL }
          catch (err) { echo "main not found, trying master…"; git branch: 'master', url: env.GITHUB_URL }
          env.GIT_COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.GIT_AUTHOR      = sh(script: 'git log -1 --pretty=%an',    returnStdout: true).trim()
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"checkout","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Detect Project Type') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"analyze","status":"running"}' || true"""
          }
          sh 'echo "📁 Files:" && ls -la'
          if (fileExists('package.json'))        { env.PROJECT_TYPE = 'nodejs'; env.PORT = '3000'; echo "🔬 Detected: Node.js" }
          else if (fileExists('requirements.txt')) { env.PROJECT_TYPE = 'python'; env.PORT = '8000'; echo "🔬 Detected: Python" }
          else if (fileExists('index.html'))     { env.PROJECT_TYPE = 'static'; env.PORT = '80';   echo "🔬 Detected: Static" }
          else                                   { env.PROJECT_TYPE = 'static'; env.PORT = '80';   echo "🔬 Unknown → Static" }
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"analyze","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Generate Dockerfile') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"prepare","status":"running"}' || true"""
          }
          if (!fileExists('Dockerfile')) {
            echo "📝 Creating Dockerfile for ${env.PROJECT_TYPE}"
            def df = (
              env.PROJECT_TYPE == 'nodejs'  ? '''FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci || npm install
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
              '''FROM nginx:alpine
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
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"prepare","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Build Image') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"build","status":"running"}' || true"""
          }
        }
        sh """
          docker build -t ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG} .
          docker tag   ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG} ${ACR_SERVER}/${PROJECT_NAME}:latest
        """
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"build","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Push to ACR') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"push","status":"running"}' || true"""
          }
        }
        withCredentials([usernamePassword(credentialsId: 'acr-credentials', usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD')]) {
          sh """
            echo "${ACR_PASSWORD}" | docker login "${ACR_SERVER}" -u "${ACR_USERNAME}" --password-stdin
            docker push ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG}
            docker push ${ACR_SERVER}/${PROJECT_NAME}:latest
          """
        }
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"push","status":"success"}' || true"""
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

    stage('Deploy to AKS (Rolling via Ingress)') {
      steps {
        withCredentials([
          file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE'),
          usernamePassword(credentialsId: 'acr-credentials', usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD')
        ]) {
          script {
            if (params.DEPLOYMENT_ID?.trim()) {
              sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"deploy","status":"running"}' || true"""
            }

            sh '''
              set -e
              export PATH="$PWD:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"

              # Namespace + imagePullSecret
              kubectl get ns "${NAMESPACE}" >/dev/null 2>&1 || kubectl create ns "${NAMESPACE}"
              kubectl -n "${NAMESPACE}" create secret docker-registry acr-auth \
                --docker-server="${ACR_SERVER}" \
                --docker-username="${ACR_USERNAME}" \
                --docker-password="${ACR_PASSWORD}" \
                --dry-run=client -o yaml | kubectl apply -f -

              # If an old Service exists and is LoadBalancer, delete it (cannot change type in-place)
              EXISTING_TYPE=$(kubectl -n "${NAMESPACE}" get svc ${PROJECT_NAME}-service -o jsonpath='{.spec.type}' 2>/dev/null || echo "")
              if [ "$EXISTING_TYPE" = "LoadBalancer" ]; then
                kubectl -n "${NAMESPACE}" delete svc ${PROJECT_NAME}-service || true
              fi

              # Deployment
              cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${PROJECT_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: 2
  strategy: { type: RollingUpdate, rollingUpdate: { maxUnavailable: 0, maxSurge: 1 } }
  selector: { matchLabels: { app: ${PROJECT_NAME} } }
  template:
    metadata: { labels: { app: ${PROJECT_NAME} } }
    spec:
      imagePullSecrets: [ { name: acr-auth } ]
      containers:
      - name: ${PROJECT_NAME}
        image: ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG}
        imagePullPolicy: IfNotPresent
        ports: [ { containerPort: ${PORT} } ]
        readinessProbe: { httpGet: { path: /, port: ${PORT} }, initialDelaySeconds: 5, periodSeconds: 5, failureThreshold: 3 }
        livenessProbe:  { httpGet: { path: /, port: ${PORT} }, initialDelaySeconds: 30, periodSeconds: 10 }
        resources:
          requests: { memory: "128Mi", cpu: "100m" }
          limits:   { memory: "512Mi", cpu: "500m" }
EOF

              # Service (ClusterIP)
              cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${PROJECT_NAME}-service
  namespace: ${NAMESPACE}
spec:
  type: ClusterIP
  selector: { app: ${PROJECT_NAME} }
  ports: [ { port: 80, targetPort: ${PORT} } ]
EOF

              # Ingress -> http://${PROJECT_NAME}.${BASE_DOMAIN}
              cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${PROJECT_NAME}-ing
  namespace: ${NAMESPACE}
spec:
  ingressClassName: nginx
  rules:
  - host: ${PROJECT_NAME}.${BASE_DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: ${PROJECT_NAME}-service, port: { number: 80 } }
EOF
            '''

            timeout(time: 10, unit: 'MINUTES') {
              sh '''
                export PATH="$PWD:$PATH"
                export KUBECONFIG="${KUBECONFIG_FILE}"
                kubectl rollout status deployment/${PROJECT_NAME} -n ${NAMESPACE} --timeout=300s
              '''
            }

            if (params.DEPLOYMENT_ID?.trim()) {
              sh """curl -s -X POST "${BACKEND_BASE}/api/internal/stages" -H "Content-Type: application/json" -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"deploy","status":"success"}' || true"""
            }
          }
        }
      }
    }

    stage('Smoke test via Ingress URL') {
      steps {
        sh '''
          URL="http://${PROJECT_NAME}.${BASE_DOMAIN}"
          echo "⏳ Waiting for ingress to route: $URL"
          sleep 25
          echo "🔎 CURL: $URL"
          set +e
          curl -I --max-time 15 "$URL" || true
          echo "LIVE: $URL"
        '''
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
          sh """curl -s -X POST "${BACKEND_BASE}/api/internal/deployments/${params.DEPLOYMENT_ID}/status" -H "Content-Type: application/json" -d '{"status":"success"}' || true"""
          sh """curl -s -X POST "${BACKEND_BASE}/api/internal/deployments/${params.DEPLOYMENT_ID}/url" -H "Content-Type: application/json" -d "{\\"external_url\\":\\"http://${PROJECT_NAME}.${BASE_DOMAIN}\\"}" || true"""
        }
      }
      echo "🎉 ✅ DEPLOYMENT SUCCESSFUL — http://${PROJECT_NAME}.${BASE_DOMAIN}"
    }
    failure {
      withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
        sh '''
          export PATH="$PWD:$PATH"
          export KUBECONFIG="${KUBECONFIG_FILE}"
          echo "---- Describe deployment ----"
          kubectl -n ${NAMESPACE} describe deploy/${PROJECT_NAME} || true
          echo "---- Ingress ----"
          kubectl -n ${NAMESPACE} describe ing/${PROJECT_NAME}-ing || true
          echo "---- Events ----"
          kubectl -n ${NAMESPACE} get events --sort-by=.lastTimestamp | tail -n 50 || true
          echo "---- Pods ----"
          kubectl -n ${NAMESPACE} get pods -o wide || true
        '''
      }
      script {
        if (params.DEPLOYMENT_ID?.trim()) {
          sh """curl -s -X POST "${BACKEND_BASE}/api/internal/deployments/${params.DEPLOYMENT_ID}/status" -H "Content-Type: application/json" -d '{"status":"failed"}' || true"""
        }
      }
      echo "💥 ❌ DEPLOYMENT FAILED"
    }
  }
}
// ------------------ end Jenkinsfile ------------------
