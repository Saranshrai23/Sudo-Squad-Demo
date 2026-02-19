pipeline {
  agent any

  environment {
    APP_NAME = "demo-nginx"
    K8S_DIR  = "k8s"
    IMAGE    = "demo-nginx:${BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Point Docker to Minikube') {
      steps {
        sh '''
          set -e
          minikube status
          eval $(minikube -p minikube docker-env)
          docker version
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -e
          eval $(minikube -p minikube docker-env)
          docker build -t demo-nginx:${BUILD_NUMBER} .
        '''
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh '''
          set -e

          # Replace placeholder with real build tag
          sed "s/BUILD_TAG/${BUILD_NUMBER}/g" ${K8S_DIR}/deployment.yaml | kubectl apply -f -
          kubectl apply -f ${K8S_DIR}/service.yaml

          kubectl rollout status deployment/demo-nginx
          kubectl get pods -o wide
          kubectl get svc demo-nginx-svc
        '''
      }
    }

    stage('Test App') {
      steps {
        sh '''
          set -e
          URL=$(minikube service demo-nginx-svc --url)
          echo "App URL: $URL"
          curl -s $URL | head -n 5
        '''
      }
    }
  }
}
