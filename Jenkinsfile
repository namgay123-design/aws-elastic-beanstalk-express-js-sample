pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()        // avoid H2 DB lock races in OWASP cache
  }

  environment {
    // === YOUR SETTINGS ===
    IMAGE_NAME      = '22261588namgayrinzin/eb-express-sample'  // your Docker Hub repo
    DOCKERHUB_CREDS = 'dockerhub'                               // your Jenkins credential ID
    // NVD API key is optional; if present use credential ID: nvd-api-key (Secret Text)
  }

  stages {

    stage('Install & Test (Node 16)') {
      steps {
        script {
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
          credentialsId: "${DOCKERHUB}",
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            set -eux
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
            docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:${BUILD_NUMBER}
            docker push ${IMAGE_NAME}:${BUILD_NUMBER}

            docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
            docker push ${IMAGE_NAME}:latest

            docker logout || true
          '''
        }
      }
    }

    stage('Dependency Scan (OWASP) - fail on High/Critical') {
      steps {
        script {
          // make NVD key optional
          def NVD_ARGS = ''
          try {
            withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
              if (env.NVD_API_KEY?.trim()) {
                echo 'Using NVD API key for faster CVE DB updates.'
                NVD_ARGS = "--nvdApiKey ${env.NVD_API_KEY}"
              }
            }
          } catch (ignored) {
            echo "NVD API key credential 'nvd-api-key' not configured. Continuing without it."
          }

          sh """
            set -eux

            # local cache and output folders
            mkdir -p odc-data odc-report
            chmod -R u+rwX odc-data odc-report || true

            # Run once, produce HTML + XML + JSON directly into workspace/odc-report
            docker run --rm \
              -v "\$PWD":/src:ro,z \
              -v "\$PWD/odc-data":/usr/share/dependency-check/data:z \
              -v "\$PWD/odc-report":/report:z \
              owasp/dependency-check:latest \
                --project "${JOB_NAME}" \
                --scan /src/package.json /src/package-lock.json \
                -f HTML -f XML -f JSON \
                -o /report \
                --exclude node_modules \
                --prettyPrint \
                --failOnCVSS 7 \
                ${NVD_ARGS}
          """
        }
      }
    }
  }

  post {
    always {
      // make it easy to debug that the report exists
      sh 'ls -lah odc-report || true'

      // keep artifacts with the build (HTML, XML, JSON)
      archiveArtifacts artifacts: 'odc-report/**', fingerprint: true, allowEmptyArchive: false

      // pretty HTML link in the build page (requires "HTML Publisher" plugin)
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

