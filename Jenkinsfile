pipeline {
  agent any

  environment {
    APP_NAME = "demo-nginx"
    K8S_DIR  = "k8s"
    IMAGE    = "demo-nginx:${BUILD_NUMBER}"
    MINIKUBE_PROFILE = "minikube"
    # For Jenkins linux agent, docker driver is most common
    MINIKUBE_DRIVER  = "docker"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Ensure Minikube Ready') {
      steps {
        sh '''
          set -e

          echo "== System check =="
          whoami
          hostname
          uname -a

          echo "== Tools check =="
          command -v docker
          command -v minikube
          command -v kubectl
          docker --version
          minikube version
          kubectl version --client=true

          echo "== Start minikube if needed =="
          # If profile doesn't exist OR cluster not running, start it
          if ! minikube profile list 2>/dev/null | grep -q "| ${MINIKUBE_PROFILE} "; then
            echo "Minikube profile '${MINIKUBE_PROFILE}' not found. Creating..."
            minikube start -p "${MINIKUBE_PROFILE}" --driver="${MINIKUBE_DRIVER}"
          else
            # profile exists; ensure it's running
            if ! minikube status -p "${MINIKUBE_PROFILE}" >/dev/null 2>&1; then
              echo "Minikube profile exists but not running. Starting..."
              minikube start -p "${MINIKUBE_PROFILE}" --driver="${MINIKUBE_DRIVER}"
            fi
          fi

          minikube status -p "${MINIKUBE_PROFILE}"

          echo "== Set kubectl context to minikube =="
          kubectl config use-context "${MINIKUBE_PROFILE}" || true
          kubectl cluster-info
        '''
      }
    }

    stage('Point Docker to Minikube') {
      steps {
        sh '''
          set -e
          eval $(minikube -p "${MINIKUBE_PROFILE}" docker-env)
          docker version
          docker info | head -n 40
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -e
          eval $(minikube -p "${MINIKUBE_PROFILE}" docker-env)

          echo "Building image: ${IMAGE}"
          docker build -t "${IMAGE}" .

          echo "Built images (filtered):"
          docker images | head -n 20
        '''
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh '''
          set -e
          kubectl config use-context "${MINIKUBE_PROFILE}" || true

          echo "Applying manifests..."
          sed "s/BUILD_TAG/${BUILD_NUMBER}/g" "${K8S_DIR}/deployment.yaml" | kubectl apply -f -
          kubectl apply -f "${K8S_DIR}/service.yaml"

          echo "Waiting for rollout..."
          kubectl rollout status deployment/"${APP_NAME}" --timeout=180s

          kubectl get pods -o wide
          kubectl get svc
          kubectl describe svc demo-nginx-svc || true
        '''
      }
    }

    stage('Test App') {
      steps {
        sh '''
          set -e

          echo "Fetching service URL..."
          URL=$(minikube -p "${MINIKUBE_PROFILE}" service demo-nginx-svc --url)
          echo "App URL: $URL"

          echo "Testing endpoint..."
          curl -fsS "$URL" | head -n 20
        '''
      }
    }
  }

  post {
    always {
      sh '''
        set +e
        echo "== Debug summary =="
        kubectl get all 2>/dev/null || true
        minikube -p "${MINIKUBE_PROFILE}" status 2>/dev/null || true
      '''
    }
  }
}
