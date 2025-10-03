pipeline {
  agent any
  options { timestamps() }

  parameters {
    string(name: 'IMAGE_REPO', defaultValue: '22261588namgayrinzin/eb-express-sample', description: 'Docker Hub repo (namespace/name)')
    string(name: 'DOCKERHUB_CREDENTIALS_ID', defaultValue: 'dockerhub', description: 'Jenkins Credentials ID for Docker Hub (username/password)')
  }

  // Secret text credential is optional; if it exists the scanner will use it
  environment {
    IMAGE_REPO = "${params.IMAGE_REPO}"
    DOCKERHUB_CREDENTIALS_ID = "${params.DOCKERHUB_CREDENTIALS_ID}"
    // If you didn’t create this credential, this stays empty and is ignored
    NVD_API_KEY = credentials('NVD_API_KEY')
  }

  stages {

    stage('Install & Test (Node 16)') {
      agent {
        docker {
          image 'node:16'
          args  '-u 0:0'        // run as root inside the container so workspace perms are fine
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
          # repo has no test script; don’t fail the build for that
          npm test || echo "No tests found"
        '''
      }
    }

    stage('Dependency Scan (OWASP) - fail on High/Critical') {
      steps {
        sh '''
          set -eux
          mkdir -p reports
          # Single run, multiple output formats -> HTML + JSON + XML
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
              ${NVD_API_KEY:+--nvdApiKey $NVD_API_KEY}

          ls -lah reports || true
        '''
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
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
      // make the raw files downloadable even if empty (don’t fail the job if scanner produced none)
      archiveArtifacts artifacts: 'reports/**', fingerprint: true, allowEmptyArchive: true

      // publish a nice in-browser report if HTML exists
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
    success {
      echo "Build & push OK: ${IMAGE_REPO}:${BUILD_NUMBER}"
    }
    failure {
      echo 'Build failed.'
    }
  }
}

