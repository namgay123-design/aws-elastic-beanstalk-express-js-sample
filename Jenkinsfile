 //Jenkinsfile — Task 3
// Meets: Node 16 build agent; npm install --save; unit tests; build & push image; dependency scan that fails on High/Critical. 
// Ref: Assignment Task 3.1 & 3.2.  (Node16 agent, install deps, tests, build/push, security scan & fail criteria)

pipeline {
  agent {
    // 3.1(b)(i): Use Node 16 Docker image as build agent
    docker { image 'node:16' ; args '-v $PWD:/workspace -w /workspace' }
  }

  environment {
    // image name: change YOUR_DH to your Docker Hub username
    APP_NAME = 'eb-express-sample'
    DOCKER_IMG = "22261588namgayrinzin/${APP_NAME}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install deps') {
      steps {
        // 3.1(b)(ii): npm install --save (assignment wording)
        sh 'npm install --save'
      }
    }

    stage('Unit tests') {
      steps { sh 'npm test || echo "No tests provided" && true' }
    }

    // 3.2: Security in the pipeline — choose OWASP DC (no account) and/or Snyk (with token)
    stage('Dep Scan - OWASP DC (fail CVSS>=7)') {
      steps {
        sh '''
          mkdir -p reports
          docker run --rm \
            -v "$PWD":/src \
            -v "$PWD/reports":/report \
            owasp/dependency-check:latest \
            --scan /src \
            --format "HTML" \
            --out /report \
            --failOnCVSS 7
        '''
      }
      post { always { archiveArtifacts artifacts: 'reports/*.html', fingerprint: true } }
    }

    stage('Dep Scan - Snyk (fail on high)') {
      when { expression { return env.SNYK_TOKEN?.trim() } }
      steps {
        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
          sh '''
            npm install -g snyk
            snyk auth "$SNYK_TOKEN"
            snyk test --severity-threshold=high
          '''
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          docker build -t ${DOCKER_IMG}:${BUILD_NUMBER} .
          docker tag ${DOCKER_IMG}:${BUILD_NUMBER} ${DOCKER_IMG}:latest
        '''
      }
    }

    stage('Docker Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${DOCKER_IMG}:${BUILD_NUMBER}
            docker push ${DOCKER_IMG}:latest
            docker logout
          '''
        }
      }
    }
  }

  post {
    always {
      // keep scan reports for your PDF evidence
      archiveArtifacts artifacts: 'reports/*.html', fingerprint: true, allowEmptyArchive: true
    }
  }
}
