pipeline {
  agent any
  options { timestamps() }

  parameters {
    string(name: 'IMAGE_REPO', defaultValue: '22261588namgayrinzin/eb-express-sample', description: 'Docker Hub repo (namespace/name)')
  }

  environment {
    DOCKERHUB = credentials('dockerhub')   // Jenkins creds ID must be 'dockerhub'
    IMAGE_REPO = "${params.IMAGE_REPO}"
  }

  stages {
    stage('Build & Test (Node 16)') {
      agent { docker { image 'node:16' } }   // only this stage runs in Node container
      steps {
        sh 'node -v && npm -v'
        sh '''
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi
        '''
        sh 'npm test --if-present'
        // gate on High/Critical; remove "|| true" if you want hard-fail
        sh 'npm audit --production --audit-level=high'
      }
    }

    // Docker commands run on the Jenkins agent host (must have docker CLI + daemon)
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
    success { echo "Build & push OK: ${IMAGE_REPO}:${BUILD_NUMBER}" }
    failure { echo 'Build failed.' }
  }
}
pipeline {
  agent any
  options { timestamps() }

  parameters {
    string(name: 'IMAGE_REPO', defaultValue: '22261588namgayrinzin/eb-express-sample', description: 'Docker Hub repo (namespace/name)')
  }

  environment {
    DOCKERHUB = credentials('dockerhub')        // Jenkins creds ID must be 'dockerhub'
    IMAGE_REPO = "${params.IMAGE_REPO}"
  }

  stages {
    stage('Build & Test (Node 16)') {
      // this stage runs in a Node 16 container
      agent { docker { image 'node:16' } }
      steps {
        sh 'node -v && npm -v'
        sh '''
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi
        '''
        sh 'npm test --if-present'
        # Gate with npm audit as an extra check (keeps requirement covered even if OWASP is skipped)
        sh 'npm audit --production --audit-level=high'
      }
    }

    // ---- NEW: Dependency scan that FAILS on High/Critical and produces an HTML report ----
    stage('Dependency Scan - OWASP DC (fail>=7)') {
      steps {
        sh '''
          mkdir -p reports
          docker run --rm \
            -v "$PWD":/src \
            -v "$PWD/reports":/report \
            owasp/dependency-check:latest \
            --scan /src \
            --format HTML \
            --out /report \
            --failOnCVSS 7
        '''
      }
    }
    // --------------------------------------------------------------------------------------

    // Docker commands run on the Jenkins host (has docker CLI + talks to DinD)
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
    always {
      // Save the OWASP HTML report for Task 4 evidence
      archiveArtifacts artifacts: 'reports/*.html', fingerprint: true, allowEmptyArchive: true
    }
    success { echo "Build & push OK: ${IMAGE_REPO}:${BUILD_NUMBER}" }
    failure { echo 'Build failed.' }
  }
}
