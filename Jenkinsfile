pipeline {
  agent any

  options { timestamps(); ansiColor('xterm') }

  parameters {
    // (Optional) lets you override from the Jenkins UI
    string(name: 'IMAGE_REPO', defaultValue: '22261588namgayrinzin/eb-express-sample', description: 'Docker Hub repo (namespace/name)')
  }

  environment {
    // Jenkins credentials ID must be 'dockerhub' (Username with password/Access Token)
    DOCKERHUB = credentials('dockerhub')
    IMAGE_REPO = "${params.IMAGE_REPO}"
  }

  stages {
    stage('Build & Test (Node 16)') {
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
        sh 'npm audit --production --audit-level=high'
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
    success { echo "Build & push OK: ${IMAGE_REPO}:${BUILD_NUMBER}" }
    failure { echo 'Build failed.' }
  }
}

