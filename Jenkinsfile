pipeline {
  // We’ll run different stages on different labeled agents
  agent none

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()           // avoid overlapping runs
    buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '20'))
  }

  triggers {
    // GitHub webhook trigger (push/PRs) – ensure webhook is configured
    githubPush()
  }

  environment {
    // Name must match the Maven tool configured in Manage Jenkins → Tools
    MAVEN_HOME = tool name: 'M3', type: 'maven'
    PATH = "${MAVEN_HOME}/bin:${env.PATH}"
    EMAIL_RECIPIENTS = 'gnanendra.mtn@gmail.com,gnan1251@gmail.com'
  }

  stages {
    stage('Checkout') {
      agent { label 'ci' }
      steps {
        checkout scm
        sh 'git --version'
        sh 'mvn -v'
      }
    }

    stage('Build & Unit Tests') {
      agent { label 'ci' }
      steps {
        sh 'mvn -B -U -Dmaven.test.failure.ignore=false clean verify'
      }
      post {
        always {
          junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
          archiveArtifacts artifacts: '**/target/*.jar,**/target/*.war', fingerprint: true, allowEmptyArchive: true
        }
      }
    }

    stage('Quality Gates (optional)') {
      when { expression { return fileExists('pom.xml') } }
      agent { label 'ci' }
      steps {
        echo 'Add static analysis / SCA / SAST here (e.g., SonarQube, OWASP dep check)'
      }
    }

    stage('Parallel Deploys') {
      // Build once, deploy to environments in parallel (change logic as you need)
      parallel {
        stage('Deploy to DEV') {
          // Runs only on branches meant for Dev (adjust to your flow)
          when {
            anyOf {
              branch 'develop'
              branch 'feature/*'
              branch 'dev'
            }
          }
          agent { label 'dev' }
          steps {
            echo "Deploying to DEV on node: ${env.NODE_NAME}"
            // Example deploy; replace with your real script or helm/k8s/ssh
            sh '''
              set -e
              echo "Pulling artifact..."
              ls -l target || true
              echo "Deploying to DEV..."
              ./deploy/dev.sh || true
            '''
          }
        }

        stage('Deploy to PROD') {
          when {
            branch 'main'
          }
          agent { label 'prod' }
          steps {
            echo "Deploying to PROD on node: ${env.NODE_NAME}"
            // Gate prod if you want a manual approval:
            // input message: 'Deploy to production?', ok: 'Deploy'
            sh '''
              set -e
              echo "Pulling artifact..."
              ls -l target || true
              echo "Deploying to PROD..."
              ./deploy/prod.sh || true
            '''
          }
        }
      }
    }
  }

  post {
    success {
      emailext(
        to: "${env.EMAIL_RECIPIENTS}",
        subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: """\
Build Succeeded ✅

Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
Branch: ${env.BRANCH_NAME ?: 'N/A'}
Commit: ${env.GIT_COMMIT ?: 'N/A'}
Triggered by: ${currentBuild?.rawBuild?.getCauseOfAction(hudson.model.Cause) ?: 'GitHub webhook'}

Artifacts: ${env.BUILD_URL}artifact/
Console:   ${env.BUILD_URL}console
""",
        mimeType: 'text/plain'
      )
    }
    failure {
      emailext(
        to: "${env.EMAIL_RECIPIENTS}",
        subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: """\
Build Failed ❌

Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
Branch: ${env.BRANCH_NAME ?: 'N/A'}
Commit: ${env.GIT_COMMIT ?: 'N/A'}

Check console output:
${env.BUILD_URL}console
""",
        mimeType: 'text/plain'
      )
    }
    unstable {
      emailext(
        to: "${env.EMAIL_RECIPIENTS}",
        subject: "UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: "Build is UNSTABLE ⚠️\n\n${env.BUILD_URL}console",
        mimeType: 'text/plain'
      )
    }
    always {
      echo "Build finished with status: ${currentBuild.currentResult}"
    }
  }
}
