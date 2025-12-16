pipeline {
  agent any

  environment {
    KUBECONFIG = credentials('kubeconfig')
    ARGOCD_NAMESPACE = "argocd"
    APP_NAMESPACE = "nginx"
    GITOPS_REPO = "https://github.com/arunprakash432/simple-helm-nginx-jenkins-project.git"
  }

  stages {

    stage('Install Helm') {
      steps {
        sh '''
          if ! command -v helm >/dev/null; then
            curl -sSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
          fi
        '''
      }
    }

    stage('Install Argo CD') {
      steps {
        sh '''
          kubectl create namespace ${ARGOCD_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

          helm repo add argo https://argoproj.github.io/argo-helm
          helm repo update

          helm upgrade --install argocd argo/argo-cd \
            --namespace ${ARGOCD_NAMESPACE} \
            --set server.service.type=LoadBalancer \
            --set configs.cm.admin\\.enabled=false \
            --wait

          kubectl wait --for=condition=Established crd/applications.argoproj.io --timeout=120s
        '''
      }
    }

    stage('Deploy NGINX via Argo CD (Helm)') {
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
              helm:
                releaseName: nginx
                valueFiles:
                  - values.yaml
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
        echo "=============================="
        echo " NGINX DEPLOYMENT SUCCESSFUL "
        echo "=============================="

        kubectl get pods -n ${APP_NAMESPACE}
        kubectl get svc  -n ${APP_NAMESPACE}
        kubectl get svc  -n ${ARGOCD_NAMESPACE}
      '''
    }
  }
}
