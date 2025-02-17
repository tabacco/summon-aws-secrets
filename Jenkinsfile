#!/usr/bin/env groovy

pipeline {
  agent { label 'executor-v2' }

  options {
    timestamps()
    buildDiscarder(logRotator(daysToKeepStr: '30'))
  }

  triggers {
    cron(getDailyCronString())
  }

  stages {

    stage('Validate') {
      parallel {
        stage('Changelog') {
          steps { sh './bin/parse-changelog.sh' }
        }
      }
    }

    stage('Build Go binaries') {
      stages {
        stage('Release artifacts') {
          when {
            buildingTag()
          }
          steps {
            sh './bin/build'
            archiveArtifacts artifacts: 'dist/*', fingerprint: true
          }
        }
        stage('Snapshot artifacts') {
          when {
            not {
              buildingTag()
            }
          }
          steps {
            sh './bin/build --snapshot'
            archiveArtifacts artifacts: 'dist/*', fingerprint: true
          }
        }
      }
    }

    stage('Run unit tests') {
      steps {
        sh './bin/test.sh'
        junit 'output/junit.xml'
        sh 'sudo chown -R jenkins:jenkins .'  // bad docker mount creates unreadable files TODO fix this
        cobertura autoUpdateHealth: true, autoUpdateStability: true, coberturaReportFile: 'output/coverage.xml', conditionalCoverageTargets: '30, 0, 0', failUnhealthy: true, failUnstable: false, lineCoverageTargets: '30, 0, 0', maxNumberOfBuilds: 0, methodCoverageTargets: '30, 0, 0', onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false
        sh 'mv output/c.out .'
        ccCoverage("gocov", "--prefix github.com/cyberark/summon-aws-secrets")
      }
    }
  }

  post {
    always {
      cleanupAndNotify(currentBuild.currentResult)
    }
  }
}
