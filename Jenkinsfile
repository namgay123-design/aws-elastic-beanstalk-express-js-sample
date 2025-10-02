pipeline {
  agent any

  options {
    timestamps()
    // Use the build wrapper directly via 'wrap' (works even when ansiColor option isn't available)
    wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm'])
  }

  parameters {
    string(name: 'IMAGE_REPO', defaultValue: '22261588namgayrinzin/eb-express-sample', description: 'Docker Hub repo (namespace/name)')
  }

  environment {
    DOCKERHUB = credentials('dockerhub')   // Jenkins creds ID must be 'dockerhub'
    IMAGE_REPO = "${params.IMAGE_REPO}"
  }

  stages {
    stage('Build & Test (Node 16)') {
      agent { docker { image 'node:16' } }   // Node steps in container
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
        // Fail on High/Critical vulns; remove this gate if it’s too strict for now
        sh 'npm audit --production --audit-level=high'
      }
    }

    // Docker runs on the Jenkins agent (must have docker CLI + daemon access)
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

