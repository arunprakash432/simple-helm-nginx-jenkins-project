pipeline {
  agent any

  environment {
    KUBECONFIG = "$HOME/.kube/config"
    ARGOCD_NAMESPACE = "argocd"
    APP_NAMESPACE = "nginx"
    GITOPS_REPO = "https://github.com/arunprakash432/simple-helm-nginx-jenkins-project.git"
  }

  stages {

    stage('Install Helm') {
      steps {
        sh '''
          if ! command -v helm >/dev/null; then
            curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
          fi
        '''
      }
    }

    stage('Install ArgoCD') {
      steps {
        sh '''
          kubectl create namespace ${ARGOCD_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

          helm repo add argo https://argoproj.github.io/argo-helm
          helm repo update

          helm upgrade --install argocd argo/argo-cd \
            --namespace ${ARGOCD_NAMESPACE} \
            --set server.service.type=LoadBalancer \
            --wait
        '''
      }
    }

    stage('Install ArgoCD CLI') {
      steps {
        sh '''
          if ! command -v argocd >/dev/null; then
            curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
            chmod +x argocd
            sudo mv argocd /usr/local/bin/
          fi
        '''
      }
    }

    stage('Deploy NGINX App via ArgoCD') {
      steps {
        sh '''
          kubectl create namespace ${APP_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

          cat <<EOF | kubectl apply -f -
          apiVersion: argoproj.io/v1alpha1
          kind: Application
          metadata:
            name: nginx
            namespace: ${ARGOCD_NAMESPACE}
          spec:
            project: default
            source:
              repoURL: ${GITOPS_REPO}
              targetRevision: HEAD
              path: nginx
            destination:
              server: https://kubernetes.default.svc
              namespace: ${APP_NAMESPACE}
            syncPolicy:
              automated:
                prune: true
                selfHeal: true
          EOF
        '''
      }
    }
  }

  post {
    success {
      sh '''
        echo "NGINX Deployment Complete"

        echo "NGINX Service:"
        kubectl get svc -n ${APP_NAMESPACE}

        echo "ArgoCD Service:"
        kubectl get svc -n ${ARGOCD_NAMESPACE}
      '''
    }
  }
}
