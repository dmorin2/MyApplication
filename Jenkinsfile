pipeline {
    agent any

    environment {
        APP_NAME = 'app'
        //APP_ARCHIVE_NAME = 'app'
        //APP_MODULE_NAME = 'android-template'
        //CHANGELOG_CMD = 'git log --date=format:"%Y-%m-%d" --pretty="format: * %s% b (%an, %cd)" | head -n 10 > commit-changelog.txt'
//        FIREBASE_GROUPS = 'mobile-dev-team, mobile-qa-team'
//        FIREBASE_APP_DIST_CMD = "firebase appdistribution:distribute app/build/outputs/apk/\$APP_BUILD_TYPE/app-\$APP_BUILD_TYPE.apk --app \$FIREBASE_ID --release-notes-file commit-changelog.txt --groups \"\$FIREBASE_GROUPS\""
//        GOOGLE_APPLICATION_CREDENTIALS = "${HOME}/google-service-accounts/${APP_MODULE_NAME}.json"
    }

    options {
        // prevent multiple builds from running concurrently, that come from the same git branch
        disableConcurrentBuilds()
    }

    stages {
        stage('Detect build type') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'develop' || env.CHANGE_TARGET == 'develop') {
                            env.BUILD_TYPE = 'debug'
                    } else if (env.BRANCH_NAME == 'master' || env.CHANGE_TARGET == 'master') {
                            env.BUILD_TYPE = 'release'
                    }
                }
            }
        }
        stage("Checkout"){
           steps {
              checkout scm
              slackSend (channel: '#jenkins-ci', color: '#48D1CC', message: "Job raised '${env.JOB_NAME} [${env.BUILD_NUMBER}]' \n More info at: (${env.BUILD_URL})")
           }
        }

        stage ("Prepare"){
           steps {
              sh 'chmod +x ./gradlew'
           }
        }
        stage('Compile') {
          steps {
            // Compile the app and its dependencies
            sh './gradlew compile${BUILD_TYPE}Sources'
          }
        }
        stages {
           stage("Build") {
              steps {
                 sh './gradlew clean assemble${BUILD_TYPE}'
              }
           }
           stage("Test") {
              steps {
                 sh './gradlew test${BUILD_TYPE}UnitTest'
              }
           }
           stage('Sonarqube') {
              environment {
                 scannerHome = tool 'SonarQubeScanner'
              }
              steps {
                 withSonarQubeEnv('sonarqube') {
                    sh './gradlew sonarqube -Dsonar.branch.name=${BRANCH_NAME}'
                 }
                 timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                 }
              }
           }
           /*stage("App Distribution") {
               steps {
                 sh './gradlew assemble${BUILD_TYPE} appDistributionUpload${BUILD_TYPE}'
               }
           }
           stage("Deploy to Play Store") {
               steps {
                 sh './gradlew publishAlphaBundle --artifact-dir app/build/outputs/bundle/alpha'
               }
           }*/
        }
        stage("Increment Version Code") {
            when {
                anyOf { branch 'master'; branch 'develop'}
            }
            steps {
                sh './gradlew incrementVersionCode'
            }
        }
    }
    post{
        always{
           echo "post-build will always run after build completed"
                // Jenkins cleans the workspace
             cleanWs()
          }
          success {
            slackSend (channel: '#jenkins-ci', color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' \n More info at: (${env.BUILD_URL})")
          }

          failure {
            slackSend (channel: '#jenkins-ci', color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' \n More info at: (${env.BUILD_URL})")
          }
        }
}