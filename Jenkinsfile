pipeline {
  agent any
  
triggers {
        pollSCM('* * * * *')
    }
  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    IMAGE_NAME           = 'reactjs-app'
    BRANCH               = "${(env.BRANCH_NAME ?: 'main').replaceAll('[^A-Za-z0-9]', '-').toLowerCase()}"
    REACT_IMAGE          = "${IMAGE_NAME}:${BRANCH}-${BUILD_NUMBER}"
    APP_PORT             = "${BRANCH == 'main' ? '8081' : '8082'}"
    COMPOSE_PROJECT_NAME                                                            = "reactjs-${BRANCH}"
  }

  stages {

    stage('1. SAST') {
      steps {
        sh '''
          pip install --user semgrep
          export PATH="$HOME/.local/bin:$PATH"
          semgrep --config auto --error .
          docker run --rm --volumes-from jenkins zricethezav/gitleaks:latest detect --source "$PWD" --no-banner
        '''
      }
    }

    stage('2. Docker build') {
      steps {
        sh ' pwd '
        sh 'docker build -t $REACT_IMAGE ./reactjs-app'
      }
    }

    stage('3. Scan image et dépendances') {
      steps {
        sh '''
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest \
            image --exit-code 0 --severity HIGH,CRITICAL $REACT_IMAGE
          docker run --rm --volumes-from jenkins owasp/dependency-check:latest \
            --scan "$PWD" --failOnCVSS 9 --noupdate || true
        '''
        sh ' pwd '
      }
    }

    stage('4. Déploiement Compose') {
      when {
        allOf {
          not { changeRequest() }
          anyOf {
            branch 'main'
            branch 'develop'
          }
        }
      }
      steps {
        sh '''
          docker compose -f docker-compose.$BRANCH.yml up -d --remove-orphans
          docker compose -f docker-compose.$BRANCH.yml ps
        '''
        sh ' pwd '
      }
    }
  }

  post {
    success { echo "Pipeline OK : $REACT_IMAGE (branche $BRANCH, port $APP_PORT)" }
    failure { echo 'Pipeline en échec, voir le stage rouge.' }
  }
}
