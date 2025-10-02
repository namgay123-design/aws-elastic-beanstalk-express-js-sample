pipeline {
  agent any

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
    DOCKERHUB = credentials('dockerhub') // Jenkins creds (Username+Password/Token)
    IMAGE_REPO = "${params.IMAGE_REPO}"
    APP_NAME   = 'aws-elastic-beanstalk-express-js-sample'
    // Optional NVD speed-up:
    // NVD_API_KEY = credentials('nvd_api_key')
  }

  stages {

    stage('Build & Test (Node 16)') {
      agent { docker { image 'node:16-alpine' } }
      steps {
        sh 'node -v && npm -v'
        sh '[ -f package-lock.json ] && npm ci || npm install --save'
        sh 'npm test --if-present'
      }
    }

    stage('OWASP Dependency-Check (fail on High/Critical)') {
      steps {
        script {
          sh 'mkdir -p reports'
          sh 'docker pull owasp/dependency-check:latest'

          // --- Enforcement: update DB into dc-data volume, reliable exit code
          def dcStatus = sh(
            returnStatus: true,
            script:
              'docker run --rm ' +
              '-v "' + env.WORKSPACE + '":/src:ro,z ' +
              '-v dc-data:/usr/share/dependency-check/data:z ' + // PERSIST DB
              'owasp/dependency-check:latest ' +
              '--project "' + env.APP_NAME + '" ' +
              '--scan /src/package.json /src/package-lock.json ' +
              '-f XML -o /tmp/dc.xml ' +          // write inside container
              '--failOnCVSS 7'
              // + ' --nvdApiKey ' + env.NVD_API_KEY  // uncomment if configured
          )

          // --- Human-readable reports, reuse cached DB (fast)
          sh 'docker run --rm ' +
             '-v "' + env.WORKSPACE + '":/src:ro,z ' +
             '-v dc-data:/usr/share/dependency-check/data:z ' +
             '-v "' + env.WORKSPACE + '/reports":/report:z ' +
             'owasp/dependency-check:latest ' +
             '--noupdate ' +
             '--project "' + env.APP_NAME + '" ' +
             '--scan /src/package.json /src/package-lock.json ' +
             '-f JSON -o /report --prettyPrint || true'

          sh 'docker run --rm ' +
             '-v "' + env.WORKSPACE + '":/src:ro,z ' +
             '-v dc-data:/usr/share/dependency-check/data:z ' +
             '-v "' + env.WORKSPACE + '/reports":/report:z ' +
             'owasp/dependency-check:latest ' +
             '--noupdate ' +
             '--project "' + env.APP_NAME + '" ' +
             '--scan /src/package.json /src/package-lock.json ' +
             '-f HTML -o /report || true'

          // Ensure we always have something to archive
          sh 'ls -lah reports || true'
          sh 'shopt -s nullglob; files=(reports/*); if [ ${#files[@]} -eq 0 ]; then echo "No report files generated. See console log." > reports/NO_REPORT.txt; fi'

          // Gate on CVSS >= 7
          if (dcStatus == 1) {
            error('OWASP DC: High/Critical vulnerabilities detected (CVSS >= 7).')
          } else if (dcStatus != 0) {
            error("OWASP DC: scanner error (exit ${dcStatus}).")
          } else {
            echo 'OWASP DC: No High/Critical detected (CVSS < 7).'
          }
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'reports/**', fingerprint: true, allowEmptyArchive: true
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
      echo "Reports archived under reports/: JSON and HTML (if generated)."
    }
    failure {
      echo 'Build failed. See console log and reports/ artifacts.'
    }
  }
}

