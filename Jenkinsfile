pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()        // avoid H2 DB lock races in OWASP cache
  }

  environment {
    IMAGE_NAME       = '22261588namgayrinzin/eb-express-sample'    // <-- your Docker Hub repo
    DOCKERHUB_CREDS  = 'dockerhub-creds'       // <-- Jenkins cred (user/pass or token)
    // The NVD API key is provided at runtime via credentials (ID: nvd-api-key)
  }

  stages {

    stage('Install & Test (Node 16)') {
      steps {
        script {
          // Runs npm steps inside a Node 16 container.
          // NOTE: Jenkins and DinD must share the jenkins-data volume so the workspace mounts correctly.
          docker.image('node:16').inside('-u root:root') {
            sh '''
              node -v
              npm install --save --no-audit --fund=false
              npm test || echo "No tests found"
            '''
          }
        }
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        withCredentials([usernamePassword(
            credentialsId: "${DOCKERHUB_CREDS}",
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
            docker tag ${IMAGE_NAME}:${BUILD_NUMBER} index.docker.io/${IMAGE_NAME}:${BUILD_NUMBER}
            docker push index.docker.io/${IMAGE_NAME}:${BUILD_NUMBER}

            docker tag ${IMAGE_NAME}:${BUILD_NUMBER} index.docker.io/${IMAGE_NAME}:latest
            docker push index.docker.io/${IMAGE_NAME}:latest
          '''
        }
      }
    }

    stage('Dependency Scan (OWASP) - fail on High/Critical') {
      steps {
        // Store your NVD key in Jenkins as a Secret Text with ID: nvd-api-key
        withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
          sh '''
            mkdir -p odc-data odc-report
            # Ensure the cache/report dirs are writable by the container user (uid 1000 is common)
            chmod -R u+rwX odc-data odc-report || true

            docker run --rm --name depcheck-$BUILD_NUMBER \
              -e NVD_API_KEY=$NVD_API_KEY \
              -v "$PWD":/src \
              -v "$PWD/odc-data":/usr/share/dependency-check/data \
              -v "$PWD/odc-report":/report \
              owasp/dependency-check:latest \
                --scan /src \
                --format HTML \
                --out /report \
                --exclude node_modules \
                --failOnCVSS 7
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'odc-report/*.html, odc-report/*.xml', fingerprint: true
    }
  }
}

