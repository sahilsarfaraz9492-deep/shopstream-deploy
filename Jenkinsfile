pipeline {
  agent any
  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'ECR image tag to deploy')
  }
  environment {
    KUBECONFIG         = '/var/lib/jenkins/.kube/shopstream-config'
    K3S_IP             = '44.192.119.64'
    AWS_DEFAULT_REGION = 'us-east-1'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
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
        withCredentials([
          string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          sh '''
            set +x
            AWS=/snap/bin/aws
            ACCOUNT=$($AWS sts get-caller-identity --query Account --output text)
            REGISTRY=$ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
            kubectl create namespace shopstream --dry-run=client -o yaml | kubectl apply -f -
            kubectl -n shopstream create secret docker-registry ecr-pull \
              --docker-server=$REGISTRY --docker-username=AWS \
              --docker-password="$($AWS ecr get-login-password)" \
              --dry-run=client -o yaml | kubectl apply -f - > /dev/null
            rm -rf rendered && mkdir rendered
            for f in k8s/*.yaml; do
              sed "s#__REGISTRY__#$REGISTRY#g; s#__TAG__#${IMAGE_TAG:-latest}#g" $f > rendered/$(basename $f)
            done
            set -x
            kubectl -n shopstream apply -f rendered/
            kubectl -n shopstream rollout restart deployment/orders deployment/users deployment/gateway
            kubectl -n shopstream rollout status deployment/orders --timeout=180s
            kubectl -n shopstream rollout status deployment/users --timeout=180s
            kubectl -n shopstream rollout status deployment/gateway --timeout=180s
          '''
        }
      }
      post {
        success { githubNotify context: 'cd/deploy', status: 'SUCCESS', description: 'Deployed to Kubernetes' }
        failure { githubNotify context: 'cd/deploy', status: 'FAILURE', description: 'Deploy or rollout failed' }
      }
    }
    stage('Verify') {
      when { branch 'main' }
      steps {
        sh '''
          for path in health api/orders api/users; do
            curl -sf --retry 10 --retry-delay 3 --retry-all-errors http://$K3S_IP/$path
            echo
          done
          kubectl -n shopstream get pods,svc
        '''
      }
      post {
        success { githubNotify context: 'cd/verify', status: 'SUCCESS', description: 'App answers through the gateway' }
        failure { githubNotify context: 'cd/verify', status: 'FAILURE', description: 'App not reachable through the gateway' }
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
