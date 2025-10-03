stage('Dependency Scan (OWASP) - fail on High/Critical') {
  steps {
    script {
      sh 'mkdir -p reports'
      def WS = pwd()  // absolute path to THIS build's workspace

      def dcCmd = """
        set -eux
        docker run --rm \\
          -v "${WS}:/src:ro" \\
          -v "dc-data:/usr/share/dependency-check/data" \\
          -v "${WS}/reports:/report" \\
          owasp/dependency-check:latest \\
            --project "${JOB_NAME}" \\
            --scan /src/package.json /src/package-lock.json \\
            -f HTML -f JSON -f XML \\
            -o /report \\
            --prettyPrint \\
            --failOnCVSS 7
      """

      if (params.NVD_CREDENTIALS_ID?.trim()) {
        withCredentials([string(credentialsId: params.NVD_CREDENTIALS_ID, variable: 'NVD_KEY')]) {
          sh "${dcCmd} --nvdApiKey \"$NVD_KEY\""
        }
      } else {
        sh dcCmd
      }
      // optional: show what was written
      sh 'ls -lah reports || true'
    }
  }
}

