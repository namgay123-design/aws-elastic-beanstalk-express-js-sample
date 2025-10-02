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
    DOCKERHUB = credentials('dockerhub') // Jenkins creds (Username+Password/Token), id: dockerhub
    IMAGE_REPO = "${params.IMAGE_REPO}"
    APP_NAME   = 'aws-elastic-beanstalk-express-js-sample'
    // Optional speed-up: add Secret Text cred 'nvd_api_key' and append
    // "--nvdApiKey ${NVD_API_KEY}" to the dependency-check commands.
    // NVD_API_KEY = credentials('nvd_api_key')
  }

  stages {

    stage('Build & Test (Node 16)') {
      agent { docker { image 'node:16-alpine' } } // run Node steps in container
      steps {
        sh 'node -v && npm -v'
        sh '[ -f package-lock.json ] && npm ci || npm install --save'
        sh 'npm test --if-present'
      }
    }

    stage('OWASP Dependency-Check (fail on High/Critical)') {
      steps {
        // Prepare + pull scanner
        sh 'mkdir -p reports'
        sh 'docker pull owasp/dependency-check:latest'

        // ENFORCEMENT RUN: reliable exit code only (no report rendering) with --disableFormat
        sh 'set +e; docker run --rm -v "$PWD":/src -v dc-data:/usr/share/dependency-check/data owasp/dependency-check:latest --project "aws-elastic-beanstalk-express-js-sample" --scan /src/package.json /src/package-lock.json --failOnCVSS 7 --disableFormat; echo $? > dc.exit; set -e'

        // BEST-EFFORT REPORTS for evidence (don’t affect pass/fail)
        sh 'docker run --rm -v "$PWD":/src -v dc-data:/usr/share/dependency-check/data -v "$PWD/reports":/report owasp/dependency-check:latest --noupdate --project "aws-elastic-beanstalk-express-js-sample" --scan /src/package.json /src/package-lock.json --format JSON --out /report --prettyPrint || true'
        sh 'docker run --rm -v "$PWD":/src -v dc-data:/usr/share/dependency-check/data -v "$PWD/reports":/report owasp/dependency-check:latest --noupdate --project "aws-elastic-beanstalk-express-js-sample" --scan /src/package.json /src/package-lock.json --format HTML --out /report || true'
        sh 'ls -lah reports || true'

        // Decide outcome from the enforcement run exit code
        sh 'code=$(cat dc.exit); if [ "$code" -eq 0 ]; then echo "OWASP DC: No High/Critical (CVSS < 7)."; elif [ "$code" -eq 1 ]; then echo "OWASP DC: High/Critical found (CVSS >= 7)."; exit 1; else echo "OWASP DC: scanner error (exit $code)."; exit $code; fi'
      }
      post {
        always {
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
        sh 'echo "${DOCKERHUB_PSW}" | docker login -u "${DOCKERHUB_USR}" --password-stdin'
        sh 'docker push ${IMAGE_REPO}:${BUILD_NUMBER}'
        sh 'docker push ${IMAGE_REPO}:latest'
        sh 'docker logout'
      }
    }
  }

  post {
    success {
      echo "Build & push OK: ${IMAGE_REPO}:${BUILD_NUMBER}"
      echo "Reports archived under reports/: JSON (robust) and HTML (if generated)."
    }
    failure {
      echo 'Build failed. See console log and reports/ artifacts.'
    }
  }
}

