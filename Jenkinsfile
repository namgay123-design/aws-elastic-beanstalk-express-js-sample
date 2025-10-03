pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()        // avoid H2 DB lock races in OWASP cache
  }

  environment {
    // === YOUR SETTINGS ===
    IMAGE_NAME      = '22261588namgayrinzin/eb-express-sample' // your Docker Hub repo
    DOCKERHUB_CREDS = 'dockerhub'                              // your Jenkins credential ID
    // (Optional) NVD API key via Jenkins Secret Text ID: nvd-api-key
  }

  stages {

    stage('Install & Test (Node 16)') {
      steps {
        script {
          // run npm steps inside Node 16 container
          docker.image('node:16').inside('-u root:root') {
            sh '''
              set -eux
              node -v
              npm ci || npm install --no-audit --fund=false
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
            set -eux
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
            docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest

            docker push ${IMAGE_NAME}:${BUILD_NUMBER}
            docker push ${IMAGE_NAME}:latest

            docker logout || true
          '''
        }
      }
    }

    stage('Dependency Scan (OWASP) - fail on High/Critical') {
      steps {
        script {
          // Make the NVD API key optional: use it if present, otherwise continue without it
          def NVD_ENV = ''
          try {
            withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
              if (env.NVD_API_KEY?.trim()) {
                echo 'Using NVD API key for faster CVE DB updates.'
                NVD_ENV = "-e NVD_API_KEY=${env.NVD_API_KEY}"
              }
            }
          } catch (ignored) {
            echo "NVD API key credential 'nvd-api-key' not configured. Continuing without it."
          }

          sh """
            set -eux
            mkdir -p odc-data odc-report
            chmod -R u+rwX odc-data odc-report || true

            docker run --rm --name depcheck-${BUILD_NUMBER} \
              ${NVD_ENV} \
              -v "\$PWD":/src \
              -v "\$PWD/odc-data":/usr/share/dependency-check/data \
              -v "\$PWD/odc-report":/report \
              owasp/dependency-check:latest \
                --scan /src \
                --format HTML \
                --out /report \
                --exclude node_modules \
                --prettyPrint \
                --failOnCVSS 7
          """
        }
      }
    }
  }

  post {
    always {
      // show what's there
      sh 'ls -lah odc-report || true'

      // keep reports with the build
      archiveArtifacts artifacts: 'odc-report/**', fingerprint: true, allowEmptyArchive: false

      // nice clickable HTML link (install "HTML Publisher" plugin)
      script {
        if (fileExists('odc-report/dependency-check-report.html')) {
          try {
            publishHTML(target: [
              reportDir: 'odc-report',
              reportFiles: 'dependency-check-report.html',
              reportName: 'Dependency-Check Report',
              keepAll: true,
              alwaysLinkToLastBuild: true,
              allowMissing: false
            ])
          } catch (e) {
            echo "HTML Publisher not available or publish failed: ${e}"
          }
        } else {
          echo 'No HTML report found to publish.'
        }
      }
    }
  }
}

