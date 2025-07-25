pipeline {
  options {
    timeout(time: 60, unit: 'MINUTES')
    ansiColor('xterm')
    disableConcurrentBuilds(abortPrevious: true)
    buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '5', numToKeepStr: '5')
  }

  agent {
    label 'linux-arm64-docker || arm64linux'
  }

  environment {
    TZ = 'UTC'
    // Amount of available vCPUs, to avoid OOM - https://www.gatsbyjs.com/docs/how-to/performance/resolving-out-of-memory-issues/#try-reducing-the-number-of-cores
    // https://github.com/jenkins-infra/jenkins-infra/tree/production/hieradata/clients/controller.ci.jenkins.io.yaml#L327
    GATSBY_CPU_COUNT = '4'
  }

  stages {
    stage('Check for typos') {
      steps {
        sh './typos --format sarif > typos.sarif || true'
      }
      post {
        always {
          recordIssues(tools: [sarif(id: 'typos', name: 'Typos', pattern: 'typos.sarif')])
        }
      }
    }

    stage('Install dependencies') {
      environment {
        NODE_ENV = 'development'
      }
      steps {
        sh '''
        asdf install
        npm install
        '''
      }
    }

    stage('Build PR') {
      when { changeRequest() }
      environment {
        NODE_ENV = 'development'
      }
      steps {
        sh '''
        npm run info
        npm run build
        '''
      }
    }

    stage('Deploy PR to preview site') {
      when {
        allOf{
          changeRequest target: 'main'
          // Only deploy from infra.ci.jenkins.io
          expression { infra.isInfra() }
        }
      }
      environment {
        NETLIFY_AUTH_TOKEN = credentials('netlify-auth-token')
      }
      steps {
        sh 'netlify-deploy --draft=true --siteName "contributor-spotlight" --title "Preview deploy for ${CHANGE_ID}" --alias "deploy-preview-${CHANGE_ID}" -d ./public'
      }
      post {
        success {
          recordDeployment('jenkins-infra', 'contributor-spotlight', pullRequest.head, 'success', "https://deploy-preview-${CHANGE_ID}--contributor-spotlight.netlify.app")
        }
        failure {
          recordDeployment('jenkins-infra', 'contributor-spotlight', pullRequest.head, 'failure', "https://deploy-preview-${CHANGE_ID}--contributor-spotlight.netlify.app")
        }
      }
    }

    stage('Deploy to production') {
      when {
        allOf{
          expression { env.BRANCH_IS_PRIMARY }
          // Only deploy from infra.ci.jenkins.io
          expression { infra.isInfra() }
        }
      }
      environment {
        NODE_ENV = 'production'
        GATSBY_MATOMO_SITE_URL = 'https://jenkins-matomo.do.g4v.dev'
        GATSBY_MATOMO_SITE_ID = '4'
      }
      steps {
        script {
          infra.withFileShareServicePrincipal([
            servicePrincipalCredentialsId: 'contributors-jenkins-io-fileshare-service-principal-writer',
            fileShare: 'contributors-jenkins-io',
            fileShareStorageAccount: 'contributorsjenkinsio'
          ]) {
            sh '''
            npm run build

            # Synchronize the File Share content
            set +x
            azcopy sync \
              --skip-version-check \
              --recursive=true \
              --delete-destination=true \
              ./public/ "${FILESHARE_SIGNED_URL}"
            '''
          }
        }
      }
    }
  }
}
