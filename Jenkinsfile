pipeline {
  agent any

  // Task 4: logging/retention
  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
  }

  parameters {
    // Set to your Docker Hub repo (you gave: 22261588namgayrinzin/eb-express-sample)
    string(name: 'IMAGE_REPO', defaultValue: '22261588namgayrinzin/eb-express-sample', description: 'Docker Hub repo (namespace/name)')
  }

  environment {
    // Jenkins creds (Username + Password/Token) with ID 'dockerhub'
    DOCKERHUB = credentials('dockerhub')
    IMAGE_REPO = "${params.IMAGE_REPO}"
    APP_NAME  = 'aws-elastic-beanstalk-express-js-sample'
  }

  stages {

    stage('Build & Test (Node 16)') {
      // Task 3.1: Use Node 16 as build agent
      agent { docker { image 'node:16-alpine' } }
      steps {
        sh 'node -v && npm -v'
        // Per brief: install deps (prefer ci if lock present, else npm install --save)
        sh '''
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install --save
          fi
        '''
        // Will be skipped if no test script exists
        sh 'npm test --if-present'
      }
    }

    stage('OWASP Dependency-Check (fail on High/Critical)') {
      // Task 3.2: Dependency vulnerability scanner
      steps {
        sh '''
          set -eux
          mkdir -p reports
          docker pull owasp/dependency-check:latest

          # Run OWASP DC against the workspace; persist NVD DB in a named volume for speed
          docker run --rm \
            -v "$PWD":/src \
            -v dc-data:/usr/share/dependency-check/data \
            -v "$PWD/reports":/report \
            owasp/dependency-check:latest \
              --project "${APP_NAME}" \
              --scan /src \
              --format "ALL" \
              --out /report \
              --failOnCVSS 7 \
              --enableExperimental
        '''
        // Task 4.2: Archive security scan artifacts (visible in build page)
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
      echo "OWASP Dependency-Check reports archived under 'reports/'."
    }
    failure {
      echo 'Build failed. Check the OWASP report (reports/dependency-check-report.html) and console log.'
    }
  }
}

