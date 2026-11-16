pipeline {
  agent any
  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'ECR image tag to deploy')
  }
  environment {
    AWS_DEFAULT_REGION    = 'us-east-1'
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    PATH                  = "/snap/bin:${env.PATH}"
    KUBECONFIG            = "${WORKSPACE}/kubeconfig-eks"
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Connect to EKS') {
      steps {
        sh '''
          aws eks update-kubeconfig --name shopstream-eks --region us-east-1 --kubeconfig "$KUBECONFIG"
          kubectl get nodes
        '''
      }
    }
    stage('Lint Manifests') {
      steps { sh 'kubectl apply --dry-run=client --validate=false -n shopstream -f k8s/' }
      post {
        success { githubNotify context: 'cd/lint', status: 'SUCCESS', description: 'Manifests are valid' }
        failure { githubNotify context: 'cd/lint', status: 'FAILURE', description: 'Kubernetes manifest has an error' }
      }
    }
    stage('Deploy') {
      when { branch 'main' }
      steps {
        sh '''
          set +x
          ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
          REGISTRY=$ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
          kubectl create namespace shopstream --dry-run=client -o yaml | kubectl apply -f -
          kubectl -n shopstream create secret docker-registry ecr-pull \
            --docker-server=$REGISTRY --docker-username=AWS \
            --docker-password="$(aws ecr get-login-password)" \
            --dry-run=client -o yaml | kubectl apply -f - > /dev/null
          rm -rf rendered && mkdir rendered
          for f in k8s/*.yaml; do
            sed "s#__REGISTRY__#$REGISTRY#g; s#__TAG__#${IMAGE_TAG:-latest}#g" $f > rendered/$(basename $f)
          done
          set -x
          kubectl -n shopstream apply -f rendered/
          kubectl -n shopstream rollout status deployment/orders --timeout=240s
          kubectl -n shopstream rollout status deployment/users --timeout=240s
          kubectl -n shopstream rollout status deployment/gateway --timeout=240s
        '''
      }
      post {
        success { githubNotify context: 'cd/deploy', status: 'SUCCESS', description: 'Deployed to EKS' }
        failure { githubNotify context: 'cd/deploy', status: 'FAILURE', description: 'Deploy or rollout failed' }
      }
    }
    stage('Verify') {
      when { branch 'main' }
      steps {
        sh '''
          HOST=""
          for i in $(seq 1 30); do
            HOST=$(kubectl -n shopstream get svc gateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
            [ -n "$HOST" ] && break
            sleep 10
          done
          echo "Load balancer address: $HOST"
          for path in health api/orders api/users; do
            curl -sf --retry 30 --retry-delay 10 --retry-all-errors http://$HOST/$path
            echo
          done
          kubectl -n shopstream get pods,svc
        '''
      }
      post {
        success { githubNotify context: 'cd/verify', status: 'SUCCESS', description: 'App answers through the load balancer' }
        failure { githubNotify context: 'cd/verify', status: 'FAILURE', description: 'App not reachable through the load balancer' }
      }
    }
  }
  post {
    failure {
      withCredentials([string(credentialsId: 'discord-webhook', variable: 'DISCORD_URL')]) {
        sh '''
          curl -s -H "Content-Type: application/json" \
            -d "{\\"content\\":\\"DEPLOY FAILED: ${JOB_NAME} #${BUILD_NUMBER} - ${BUILD_URL}\\"}" \
            "$DISCORD_URL"
        '''
      }
    }
  }
}
