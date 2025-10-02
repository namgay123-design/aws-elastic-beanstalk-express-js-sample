pipeline {
  agent any

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20')) // Task 4
  }

  parameters {
    string(name: 'IMAGE_REPO', defaultValue: '22261588namgayrinzin/eb-express-sample', description: 'Docker Hub repo (namespace/name)')
  }

  environment {
    DOCKERHUB = credentials('dockerhub') // Jenkins creds (Username+Password/Token) id: dockerhub
    IMAGE_REPO = "${params.IMAGE_REPO}"
    APP_NAME   = 'aws-elastic-beanstalk-express-js-sample'
    // Optional: set this in Jenkins as a Secret Text credential and switch the line in OWASP stage
    // NVD_API_KEY = credentials('nvd_api_key')
  }

  stages {

    stage('Build & Test (Node 16)') {
      agent { docker { image 'node:16-alpine' } }
      steps {
        sh 'node -v && npm -v'
        sh '''
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install --save
          fi
        '''
        sh 'npm test --if-present'
      }
    }

    stage('OWASP Dependency-Check (fail on High/Critical)') {
      steps {
        sh '''
          set -eux
          mkdir -p reports
          docker pull owasp/dependency-check:latest

          # Scan only the Node lockfiles; avoid node_modules requirement and noisy warnings
          docker run --rm \
            -v "$PWD":/src \
            -v dc-data:/usr/share/dependency-check/data \
            -v "$PWD/reports":/report \
            owasp/dependency-check:latest \
              --project "${APP_NAME}" \
              --scan /src/package.json /src/package-lock.json \
              --format HTML \
              --out /report \
              --failOnCVSS 7
              # If you create a Secret Text credential 'nvd_api_key', add:
              # --nvdApiKey ${NVD_API_KEY}
        '''
        archiveArtifacts artifacts: 'reports/**', fingerprint: true
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker version'
        sh 'docker build -t ${IMAGE_REPO}:${BUILD_NUMBER} -t ${IMAGE_REPO}:latest .'
      }
    }

    stage('Docker Push') {
      steps {
        sh '''
          echo "${DOCKERHUB_PSW}" | docker login -u "${DOCKERHUB_USR}" --password-stdin
          docker push ${IMAGE_REPO}:${BUILD_NUMBER}
          docker push ${IMAGE_REPO}:latest
          docker logout
        '''
      }
    }
  }

  post {
    success {
      echo "Build & push OK: ${IMAGE_REPO}:${BUILD_NUMBER}"
      echo "OWASP report archived: reports/dependency-check-report.html"
    }
    failure {
      echo 'Build failed. See console and reports/dependency-check-report.html'
    }
  }
}

