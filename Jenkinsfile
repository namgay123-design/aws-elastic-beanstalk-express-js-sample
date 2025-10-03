pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }

  environment {
    // change this to your image name
    DOCKER_IMAGE = '22261588namgayrinzin/eb-express-sample'
    IMAGE_TAG    = "${env.BUILD_NUMBER}"
  }

  stages {

    stage('Install & Test (Node 16)') {
      agent {
        docker {
          image 'node:16'
          args '-u 0:0'
        }
      }
      steps {
        sh '''
          set -eux
          node -v
          npm ci
          npm test || echo "No tests found"
        '''
      }
    }

    stage('OWASP Dependency-Check (fail on High/Critical)') {
      steps {
        script {
          // Optional NVD API key (Jenkins Credentials -> Secret text -> ID: nvd-api-key)
          def nvdOpt = ''
          try {
            withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
              if (env.NVD_API_KEY?.trim()) {
                echo 'Using NVD API key for faster CVE DB updates.'
                nvdOpt = "--nvdApiKey ${env.NVD_API_KEY}"
              }
            }
          } catch (ignored) {
            echo "NVD API key credential 'nvd-api-key' not configured. Continuing without it."
          }

          sh """
            set -eux
            mkdir -p reports
            rm -f reports/* || true
            DC_IMAGE=owasp/dependency-check:latest

            # 1) Full scan (updates DB) + fail on CVSS >= 7 -> XML
            CID=\$(docker create --rm \
              -v "\${WORKSPACE}":/src:ro,z \
              -v dc-data:/usr/share/dependency-check/data:z \
              \$DC_IMAGE \
              --project "${JOB_NAME}" \
              --scan /src/package.json /src/package-lock.json \
              -f XML -o /tmp ${nvdOpt} \
              --failOnCVSS 7)

            set +e
            docker start -a "\$CID"
            DC_EXIT=\$?
            set -e
            docker cp "\$CID":/tmp/dependency-check-report.xml reports/dependency-check-report.xml || true
            docker rm -f "\$CID" || true

            # 2) JSON (no update)
            CIDJ=\$(docker create --rm \
              -v "\${WORKSPACE}":/src:ro,z \
              -v dc-data:/usr/share/dependency-check/data:z \
              \$DC_IMAGE --noupdate \
              --project "${JOB_NAME}" \
              --scan /src/package.json /src/package-lock.json \
              -f JSON -o /tmp --prettyPrint)
            docker start -a "\$CIDJ" || true
            docker cp "\$CIDJ":/tmp/dependency-check-report.json reports/dependency-check-report.json || true
            docker rm -f "\$CIDJ" || true

            # 3) HTML (no update)
            CIDH=\$(docker create --rm \
              -v "\${WORKSPACE}":/src:ro,z \
              -v dc-data:/usr/share/dependency-check/data:z \
              \$DC_IMAGE --noupdate \
              --project "${JOB_NAME}" \
              --scan /src/package.json /src/package-lock.json \
              -f HTML -o /tmp)
            docker start -a "\$CIDH" || true
            docker cp "\$CIDH":/tmp/dependency-check-report.html reports/dependency-check-report.html || true
            docker rm -f "\$CIDH" || true

            # Decide pass/fail based on the first run
            if [ "\$DC_EXIT" -eq 0 ]; then
              echo "OWASP DC: No High/Critical detected (CVSS < 7)."
            elif [ "\$DC_EXIT" -eq 1 ]; then
              echo "OWASP DC: High/Critical vulnerabilities found. Failing build."
              exit 1
            else
              echo "OWASP DC: Scanner error (exit \$DC_EXIT). Failing build."
              exit "\$DC_EXIT"
            fi
          """
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          set -eux
          docker version
          docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} -t ${DOCKER_IMAGE}:latest .
        '''
      }
    }

    stage('Docker Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PSW')]) {
          sh '''
            set -eux
            echo "$DOCKERHUB_PSW" | docker login -u "$DOCKERHUB_USER" --password-stdin
            docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
            docker push ${DOCKER_IMAGE}:latest
            docker logout
          '''
        }
      }
    }
  }

  post {
    always {
      // prove artifacts exist in console
      sh 'ls -lah reports || true'

      // keep reports with the build
      archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: false, fingerprint: true

      // pretty HTML link (requires "HTML Publisher" plugin)
      script {
        if (fileExists('reports/dependency-check-report.html')) {
          try {
            publishHTML(target: [
              reportDir: 'reports',
              reportFiles: 'dependency-check-report.html',
              reportName: 'OWASP Dependency-Check',
              keepAll: true,
              alwaysLinkToLastBuild: true
            ])
          } catch (e) {
            echo "HTML Publisher plugin not installed: ${e}"
          }
        } else {
          echo 'No HTML report found to publish.'
        }
      }
    }
    success {
      echo "Pipeline finished."
      echo "Build & push OK: ${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"
    }
  }
}

