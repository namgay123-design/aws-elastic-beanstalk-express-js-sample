pipeline {
  agent any
  options { timestamps() }

  parameters {
    string(name: 'IMAGE_REPO',
           defaultValue: '22261588namgayrinzin/eb-express-sample',
           description: 'Docker Hub repo (namespace/name)')
    booleanParam(name: 'USE_NVD_KEY',
                 defaultValue: false,
                 description: 'Use Jenkins Secret Text (ID: NVD_API_KEY) to speed Dependency-Check updates')
  }

  environment {
    IMAGE_REPO = "${params.IMAGE_REPO}"
  }

  stages {

    stage('Build & Test (Node 16)') {
      agent { docker { image 'node:16-alpine' } }
      steps {
        sh 'node -v && npm -v'
        sh '''
          set -eux
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi
        '''
        sh 'npm test --if-present'
      }
    }

    stage('OWASP Dependency-Check (fail on High/Critical)') {
      steps {
        script {
          // Prepare reports dir
          sh '''
            set -eux
            mkdir -p reports
            rm -f reports/*
          '''

          // Reusable shell body that:
          // 1) seeds/updates DC DB (XML output to container /tmp, captures exit code)
          // 2) writes JSON + HTML reports into container /tmp and copies to ./reports
          def scanScript = '''
            set -eux

            # Compose optional NVD API flag if variable is present
            NVD_OPT=""
            if [ "${USE_NVD_KEY:-false}" = "true" ] && [ -n "${NVD_API_KEY:-}" ]; then
              NVD_OPT="--nvdApiKey ${NVD_API_KEY}"
            fi

            # 1) Seed/update DB and gate on CVSS >= 7
            CID=$(docker create --rm \
              -v "$WORKSPACE":/src:ro,z \
              -v dc-data:/usr/share/dependency-check/data:z \
              owasp/dependency-check:latest \
              --project "aws-elastic-beanstalk-express-js-sample" \
              --scan /src/package.json /src/package-lock.json \
              -f XML -o /tmp \
              --failOnCVSS 7 ${NVD_OPT})

            set +e
            docker start -a "$CID"
            DC_EXIT=$?
            set -e

            # Copy XML report out (if it exists)
            docker cp "$CID":/tmp/dependency-check-report.xml reports/dependency-check-report.xml 2>/dev/null || true
            docker rm -f "$CID" >/dev/null 2>&1 || true

            # 2) Create JSON report (reusing DB, no update)
            CID_J=$(docker create --rm \
              -v "$WORKSPACE":/src:ro,z \
              -v dc-data:/usr/share/dependency-check/data:z \
              owasp/dependency-check:latest \
              --noupdate \
              --project "aws-elastic-beanstalk-express-js-sample" \
              --scan /src/package.json /src/package-lock.json \
              -f JSON -o /tmp --prettyPrint)
            docker start -a "$CID_J" || true
            docker cp "$CID_J":/tmp/dependency-check-report.json reports/dependency-check-report.json 2>/dev/null || true
            docker rm -f "$CID_J" >/dev/null 2>&1 || true

            # 3) Create HTML report (reusing DB, no update)
            CID_H=$(docker create --rm \
              -v "$WORKSPACE":/src:ro,z \
              -v dc-data:/usr/share/dependency-check/data:z \
              owasp/dependency-check:latest \
              --noupdate \
              --project "aws-elastic-beanstalk-express-js-sample" \
              --scan /src/package.json /src/package-lock.json \
              -f HTML -o /tmp)
            docker start -a "$CID_H" || true
            docker cp "$CID_H":/tmp/dependency-check-report.html reports/dependency-check-report.html 2>/dev/null || true
            docker rm -f "$CID_H" >/dev/null 2>&1 || true

            echo "$DC_EXIT" > reports/dc.exit
            exit "$DC_EXIT"
          '''

          int dcExit = 0
          if (params.USE_NVD_KEY) {
            withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
              // pass the toggle to the shell as well
              withEnv(["USE_NVD_KEY=true"]) {
                dcExit = sh(returnStatus: true, script: scanScript)
              }
            }
          } else {
            withEnv(["USE_NVD_KEY=false"]) {
              dcExit = sh(returnStatus: true, script: scanScript)
            }
          }

          // Decide build result from exit code
          if (dcExit == 1) {
            error "OWASP Dependency-Check: High/Critical vulnerabilities found (CVSS >= 7). See reports/."
          } else if (dcExit != 0) {
            error "OWASP Dependency-Check: scanner error (exit ${dcExit}). See console log."
          } else {
            echo "OWASP DC: No High/Critical detected (CVSS < 7)."
          }
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true, fingerprint: true
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
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKERHUB_USR', passwordVariable: 'DOCKERHUB_PSW')]) {
          sh '''
            set -eux
            echo "${DOCKERHUB_PSW}" | docker login -u "${DOCKERHUB_USR}" --password-stdin
            docker push ${IMAGE_REPO}:${BUILD_NUMBER}
            docker push ${IMAGE_REPO}:latest
            docker logout
          '''
        }
      }
    }
  }

  post {
    success { echo "Build & push OK: ${IMAGE_REPO}:${BUILD_NUMBER}" }
    failure { echo 'Build failed. See console and the reports/ artifacts if produced.' }
    always  { echo 'Pipeline finished.' }
  }
}

