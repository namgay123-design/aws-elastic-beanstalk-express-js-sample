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
      }
    }

    stage('Dependency Scan (npm audit)') {
      agent { docker { image 'node:16' } }
      steps {
        sh '''
          # Ensure deps are present (safe to run again)
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi

          # Always produce a report; then we decide if we fail
          npm audit --production --json > audit.json || true

          node -e "
            const fs=require('fs');
            const j=JSON.parse(fs.readFileSync('audit.json','utf8'));
            let hi=0, cr=0;
            if (j.vulnerabilities){ hi=j.vulnerabilities.high||0; cr=j.vulnerabilities.critical||0; }
            else if (j.metadata && j.metadata.vulnerabilities){ hi=j.metadata.vulnerabilities.high||0; cr=j.metadata.vulnerabilities.critical||0; }
            console.log('High:',hi,'Critical:',cr);
            if ((hi+cr)>0){ console.error('Failing due to High/Critical vulnerabilities'); process.exit(1); }
          "
        '''
        archiveArtifacts artifacts: 'audit.json', fingerprint: true
      }
    }

    // Docker commands run on the Jenkins agent host (must have docker CLI + daemon)
    stage('Docker Build') {
      steps {
        sh 'docker version'
        sh 'docker build -t ${IMAGE_REPO}:${BUILD_NUMBER} -t ${IMAGE_REPO}:latest .'
      }
    }

    // Scan the built image BEFORE pushing it
    stage('Container Image Scan (Trivy)') {
      steps {
        sh '''
          # Pull Trivy image and scan High/Critical; fail build if found
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:0.54.2 image \
            --severity HIGH,CRITICAL \
            --exit-code 1 \
            --no-progress \
            -f table \
            -o trivy-image.txt \
            ${IMAGE_REPO}:${BUILD_NUMBER}
        '''
        archiveArtifacts artifacts: 'trivy-image.txt', fingerprint: true
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

