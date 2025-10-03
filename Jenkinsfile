pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds() // avoid H2 DB lock races in OWASP cache
  }

  environment {
    IMAGE_NAME      = '22261588namgayrinzin/eb-express-sample' // your Docker Hub repo
    DOCKERHUB_CREDS = 'dockerhub'                              // your Jenkins credential ID
    // Optional: add a Secret Text in Jenkins with ID 'nvd-api-key' for much faster updates
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

    stage('Dependency Scan (OWASP) - fast') {
      steps {
        script {
          // Try to use NVD API key if configured
          def NVD_ENV = ''
          try {
            withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
              if (env.NVD_API_KEY?.trim()) {
                echo 'Using NVD API key for faster NVD updates.'
                NVD_ENV = "-e NVD_API_KEY=${env.NVD_API_KEY}"
              }
            }
          } catch (ignored) {
            echo "NVD API key credential 'nvd-api-key' not configured. Continuing without it."
          }

          // Decide if we can skip updates (cache fresh within 24h)
          def NOUPDATE = ''
          sh 'mkdir -p odc-data odc-report'
          def cacheFresh = sh(
            script: 'if find odc-data -type f -mtime -1 | grep -q .; then echo fresh; else echo stale; fi',
            returnStdout: true
          ).trim()
          if (cacheFresh == 'fresh') {
            NOUPDATE = '--noupdate'
            echo "ODC cache looks fresh; scanning with --noupdate."
          } else {
            echo "ODC cache stale or empty; will update CVE DB this run."
          }

          // Pre-pull to avoid first-run delay
          sh 'docker pull owasp/dependency-check:latest || true'

          // FAST scan: only the manifests (package.json + package-lock.json)
          // This still resolves/transitively analyzes dependencies via the lock file.
          sh """
            set -eux
            chmod -R u+rwX odc-data odc-report || true

            docker run --rm --name depcheck-${BUILD_NUMBER} \
              ${NVD_ENV} \
              -v "\$PWD":/src \
              -v "\$PWD/odc-data":/usr/share/dependency-check/data \
              -v "\$PWD/odc-report":/report \
              owasp/dependency-check:latest \
                ${NOUPDATE} \
                --scan /src/package.json /src/package-lock.json \
                --format HTML \
                --out /report \
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

      // clickable HTML link (requires "HTML Publisher" plugin)
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

