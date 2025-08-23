pipeline {
  agent any

  parameters {
    string(name: 'GITHUB_URL',   defaultValue: 'https://github.com/you/your-repo', description: 'GitHub repository URL')
    string(name: 'PROJECT_NAME', defaultValue: 'myapp', description: 'Short project slug (used for ACR repo, K8s app name)')
    string(name: 'DEPLOYMENT_ID', defaultValue: '', description: 'Optional deployment ID for callbacks to backend')
    string(name: 'LB_RG', defaultValue: 'MC_devops-monitoring-rg_devops-aks_eastus', description: 'AKS managed RG')
    string(name: 'LB_IP', defaultValue: '', description: 'Optional static public IP from Azure (empty = AKS allocates)')
  }

  environment {
    ACR_SERVER   = 'devopsmonitoracrrt2y5a.azurecr.io'
    IMAGE_TAG    = "${BUILD_NUMBER}"
    NAMESPACE    = "${params.PROJECT_NAME}-dev"
    BACKEND_BASE = 'http://localhost:4000'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: "${params.GITHUB_URL}"
      }
    }

    stage('Build Docker Image') {
      steps {
        sh """
          docker build -t ${ACR_SERVER}/${params.PROJECT_NAME}:${IMAGE_TAG} .
          docker tag ${ACR_SERVER}/${params.PROJECT_NAME}:${IMAGE_TAG} ${ACR_SERVER}/${params.PROJECT_NAME}:latest
        """
      }
    }

    stage('Push to ACR') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'acr-credentials',
                                          usernameVariable: 'ACR_USERNAME',
                                          passwordVariable: 'ACR_PASSWORD')]) {
          sh """
            echo "${ACR_PASSWORD}" | docker login ${ACR_SERVER} -u ${ACR_USERNAME} --password-stdin
            docker push ${ACR_SERVER}/${params.PROJECT_NAME}:${IMAGE_TAG}
            docker push ${ACR_SERVER}/${params.PROJECT_NAME}:latest
          """
        }
      }
    }

    stage('Deploy to AKS') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=${KUBECONFIG_FILE}
            kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}
            kubectl apply -f k8s.yaml
          '''
        }
      }
    }
  }
}
