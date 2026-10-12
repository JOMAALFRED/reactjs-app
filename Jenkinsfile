pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    HARBOR_URL     = 'harbor.local'
    HARBOR_PROJECT = 'devsecops'
    IMAGE_NAME     = 'app'
    IMAGE          = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${BUILD_NUMBER}"

    REACT_IMAGE_NAME = 'reactjs-app'
    REACT_IMAGE      = "${HARBOR_URL}/${HARBOR_PROJECT}/${REACT_IMAGE_NAME}:${BUILD_NUMBER}"
  }

  stages {

    stage('1. SAST') {
      steps {
        sh '''
          pip install --user bandit semgrep
          export PATH="$HOME/.local/bin:$PATH"
          bandit -r . -ll -x ./tests,./.git
          semgrep --config auto --error .
          docker run --rm --volumes-from jenkins zricethezav/gitleaks:latest detect --source "$PWD" --no-banner
        '''
      }
    }

    stage('2. Docker build') {
      steps {
        sh 'docker build -t $IMAGE .'
      }
    }

    stage('2b. Docker build React') {
      steps {
        sh 'docker build -t $REACT_IMAGE ./reactjs-app'
      }
    }

    stage('3. Scan image et dépendances') {
      steps {
        sh '''
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest \
            image --exit-code 0 --severity HIGH,CRITICAL $IMAGE
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest \
            image --exit-code 0 --severity HIGH,CRITICAL $REACT_IMAGE
          docker run --rm --volumes-from jenkins owasp/dependency-check:latest \
            --scan "$PWD" --failOnCVSS 9 --noupdate || true
        '''
      }
    }

    stage('4. Push Harbor') {
      when { expression { false } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds',
                          usernameVariable: 'H_USER', passwordVariable: 'H_PASS')]) {
          sh '''
            echo "$H_PASS" | docker login $HARBOR_URL -u "$H_USER" --password-stdin
            docker push $IMAGE
            docker push $REACT_IMAGE
          '''
        }
      }
    }

    stage('5. Signature Cosign') {
      when { expression { false } }
      steps {
        withCredentials([file(credentialsId: 'cosign-key', variable: 'COSIGN_KEY'),
                         string(credentialsId: 'cosign-password', variable: 'COSIGN_PASSWORD')]) {
          sh '''
            cosign sign --yes --key $COSIGN_KEY $IMAGE
            cosign sign --yes --key $COSIGN_KEY $REACT_IMAGE
          '''
        }
      }
    }

    stage('6. Policy check Harbor') {
      when { expression { false } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds',
                          usernameVariable: 'H_USER', passwordVariable: 'H_PASS'),
                         file(credentialsId: 'cosign-pub', variable: 'COSIGN_PUB')]) {
          sh '''
            cosign verify --key $COSIGN_PUB $IMAGE
            curl -sf -u "$H_USER:$H_PASS" \
              "https://$HARBOR_URL/api/v2.0/projects/$HARBOR_PROJECT/repositories/$IMAGE_NAME/artifacts/$BUILD_NUMBER" \
              | tee /tmp/artifact.json | grep -q '"digest"'
          '''
        }
      }
    }

    stage('7. Déploiement Compose') {
      steps {
        sh '''
          export IMAGE=$IMAGE
          export REACT_IMAGE=$REACT_IMAGE
          docker compose up -d --remove-orphans
          docker compose ps
        '''
      }
    }
  }

  post {
    always  { sh 'docker logout $HARBOR_URL || true' }
    success { echo "Pipeline OK : $IMAGE et $REACT_IMAGE (React sur http://localhost:8081)" }
    failure { echo 'Pipeline en échec, voir le stage rouge.' }
  }
}
