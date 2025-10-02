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
    // Jenkins credentials (Username + Password/Token) with ID 'dockerhub'
    DOCKERHUB = credentials('dockerhub')
    IMAGE_REPO = "${params.IMAGE_REPO}"
    APP_NAME   = 'aws-elastic-beanstalk-express-js-sample'
    // Optional speed-up: add a Secret Text cred 'nvd_api_key' and uncomment below + CLI flag
    // NVD_API_KEY = credentials('nvd_api_key')
  }

  stages {
    stage('Build & Test (Node 16)') {
      agent { docker { image 'node:16-alpine' } }   // Node steps in a container
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

          # --- Enforcement run: XML report + gate on CVSS >= 7 ---
          set +e
          docker run --rm \
            -v "$PWD":/src \
            -v dc-data:/usr/share/dependency-check/data \
            -v "$PWD/reports":/report \
            owasp/dependency-check:latest \
              --project "${APP_NAME}" \
              --scan /src/package.json /src/package-lock.json \
              --format XML \
              --out /report \
              --failOnCVSS 7
              # --nvdApiKey ${NVD_API_KEY}   # uncomment if you configured NVD_API_KEY
          DC_EXIT=$?
          set -e

          # --- Best-effort HTML (for humans). Do not affect gate/exit. ---
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

          # --- Decide outcome based on enforcement run exit code ---
          if [ "$DC_EXIT" -eq 0 ]; then
            echo "OWASP DC: No High/Critical detected (CVSS < 7)."
          elif [ "$DC_EXIT" -eq 1 ]; then
            echo "OWASP DC: High/Critical vulnerabilities detected (CVSS >= 7). Failing build."
            exit 1
          else
            echo "OWASP DC: scanner error (exit $DC_EXIT). Failing build."
            exit $DC_EXIT
          fi
        '''
      }
      post {
        always {
          // Task 4: archive reports regardless of pass/fail so the marker can open them
          archiveArtifacts artifacts: 'reports/**', fingerprint: true
        }
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
      echo "Reports archived: reports/dependency-check-report.xml (enforcement), dependency-check-report.html (view)"
    }
    failure {
      echo 'Build failed. See console log and the Dependency-Check reports under artifacts.'
    }
  }
}

