pipeline {
  agent any

  // Task 4: timestamps + retention
  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
  }

  parameters {
    string(
      name: 'IMAGE_REPO',
      defaultValue: '22261588namgayrinzin/eb-express-sample',
      description: 'Docker Hub repo (namespace/name)'
    )
  }

  environment {
    DOCKERHUB = credentials('dockerhub')     // Jenkins creds (Username+Password/Token)
    IMAGE_REPO = "${params.IMAGE_REPO}"
    APP_NAME   = 'aws-elastic-beanstalk-express-js-sample'
    // Optional speed-up:
    // NVD_API_KEY = credentials('nvd_api_key')
  }

  stages {
    stage('Build & Test (Node 16)') {
      agent { docker { image 'node:16-alpine' } }   // Only this stage runs in Node container
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

          # -------- Enforcement: no report, just reliable exit code on CVSS >= 7 --------
          set +e
          docker run --rm \
            -v "$PWD":/src \
            -v dc-data:/usr/share/dependency-check/data \
            owasp/dependency-check:latest \
              --project "${APP_NAME}" \
              --scan /src/package.json /src/package-lock.json \
              --failOnCVSS 7 \
              --disableFormat
              # --nvdApiKey ${NVD_API_KEY}
          DC_EXIT=$?
          set -e

          # -------- Best-effort reports for evidence (don’t affect the gate) --------
          docker run --rm \
            -v "$PWD":/src \
            -v dc-data:/usr/share/dependency-check/data \
            -v "$PWD/reports":/report \
            owasp/dependency-check:latest \
              --noupdate \
              --project "${APP_NAME}" \
              --scan /src/package.json /src/package-lock.json \
              --format JSON \
              --out /report \
              --prettyPrint || true

          docker run --rm \
            -v "$PWD":/src \
            -v dc-data:/usr/share/dependency-check/data \
            -v "$PWD/reports":/report \
            owasp/dependency-check:latest \
              --noupdate \
              --project "${APP_NAME}" \
              --scan /src/package.json /src/package-lock.json \
              --format HTML \
              --out /report || true

          ls -lah reports || true

          # -------- Decide outcome using en

