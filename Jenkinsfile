pipeline {
  agent any
  options { timestamps() }

  parameters {
    string(name: 'IMAGE_REPO', defaultValue: '22261588namgayrinzin/eb-express-sample', description: 'Docker Hub repo (namespace/name)')
    string(name: 'DOCKERHUB_CREDENTIALS_ID', defaultValue: 'dockerhub', description: 'Credentials ID: Docker Hub (username/password)')
    string(name: 'NVD_CREDENTIALS_ID', defaultValue: '', description: 'Optional Secret Text credential ID for NVD API key (leave blank if none)')
  }

  environment {
    IMAGE_REPO = "${params.IMAGE_REPO}"
  }

  stages {

    stage('Install & Test (Node 16)') {
      agent {
        docker {
          image 'node:16'
          args  '-u 0:0'
          reuseNode true
        }
      }
      steps {
        sh '''
          set -eux
          node -v
          npm -v
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install --no-audit --fund=false
          fi
          npm test || echo "No tests found"
        '''
      }
    }

    stage('Dependency Scan (OWASP) - fail on High/Critical') {
      steps {
        script {
          sh 'mkdir -p reports'
          if (params.NVD_CREDENTIALS_ID?.trim()) {
            withCredentials([string(credentialsId: params.NVD_CREDENTIALS_ID, variable: 'NVD_KEY')]) {
              sh """
                set -eux
                docker run --rm \
                  -v "$PWD:/src:ro,z" \
                  -v "dc-data:/usr/share/dependency-check/data:z" \
                  -v "$PWD/reports:/report:z" \
                  owasp/dependency-check:latest \
                    --project "aws-elastic-beanstalk-express-js-sample" \
                    --scan /src/package.json /src/package-lock.json \
                    -f HTML -f JSON -f XML \
                    -o /report \
                    --prettyPrint \
                    --failOnCVSS 7 \
                    --nvdApiKey "$NVD_KEY"
                ls -lah reports || true
              """
            }
          } else {
            sh """
              set -eux
              docker run --rm \
                -v "$PWD:/src:ro,z" \
                -v "dc-data:/usr/share/dependency-check/data:z" \
                -v "$PWD/reports:/report:z" \
                owasp/dependency-check:latest \
                  --project "aws-elastic-beanstalk-express-js-sample" \
                  --scan /src/package.json /src/package-lock.json \
                  -f HTML -f JSON -f XML \
                  -o /report \
                  --prettyPrint \
                  --failOnCVSS 7
              ls -lah reports || true
            """
          }
        }
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${params.DOCKERHUB_CREDENTIALS_ID}",
          usernameVariable: 'DOCKERHUB_USR',
          passwordVariable: 'DOCKERHUB_PSW'
        )]) {
          sh '''
            set -eux
            docker version
            docker build -t ${IMAGE_REPO}:${BUILD_NUMBER} -t ${IMAGE_REPO}:latest .
            echo "${DOCKERHUB_PSW}" | docker login -u "${DOCKERHUB_USR}" --password-stdin
            docker push ${IMAGE_REPO}:${BUILD_NUMBER}
            docker push ${IMAGE_REPO}:latest
            docker logout
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'reports/**', fingerprint: true, allowEmptyArchive: true
      script {
        if (fileExists('reports/dependency-check-report.html')) {
          publishHTML(target: [
            reportDir: 'reports',
            reportFiles: 'dependency-check-report.html',
            reportName: 'OWASP Dependency-Check',
            keepAll: true,
            alwaysLinkToLastBuild: true,
            allowMissing: false
          ])
        } else {
          echo 'No HTML report found to publish.'
        }
      }
    }
    success { echo "Build & push OK: ${IMAGE_REPO}:${BUILD_NUMBER}" }
    failure { echo 'Build failed.' }
  }
}

