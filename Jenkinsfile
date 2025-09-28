pipeline {
  agent { label 'docker-agent1' }   // <-- your external agent's label

  options { timestamps(); skipDefaultCheckout(true) }

  environment {
    IMAGE            = 'nelldocker247/firstjenkins'      // Docker Hub repo
    DOCKERHUB_CREDS  = 'dockerhub-credentials-id'        // Jenkins creds ID (username+token)
    DOCKER_BUILDKIT  = '1'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD > .git/shortsha'
      }
    }

    stage('Preflight') {
      steps { sh 'whoami && docker version && docker info | head -n 20' }
    }

    stage('Build') {
      steps {
        script { env.SHORT_SHA = readFile('.git/shortsha').trim() }
        withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDS}",
                                          usernameVariable: 'U',
                                          passwordVariable: 'P')]) {
          sh '''
            echo "$P" | docker login -u "$U" --password-stdin
            docker build --pull -t ${IMAGE}:${SHORT_SHA} .
            docker logout || true
          '''
        }
      }
    }

    stage('Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDS}",
                                          usernameVariable: 'U',
                                          passwordVariable: 'P')]) {
          sh '''
            echo "$P" | docker login -u "$U" --password-stdin

            # push commit tag
            docker push ${IMAGE}:${SHORT_SHA}

            # also push :latest on main
            CURRENT_BRANCH="${BRANCH_NAME:-${GIT_BRANCH:-main}}"
            if [ "$CURRENT_BRANCH" = "main" ] || [ "$CURRENT_BRANCH" = "master" ] || [ "$CURRENT_BRANCH" = "refs/heads/main" ]; then
              docker tag ${IMAGE}:${SHORT_SHA} ${IMAGE}:latest
              docker push ${IMAGE}:latest
            fi

            docker logout || true
          '''
        }
      }
    }
  }

  post {
    always { sh 'docker image prune -f || true' }
  }
}
