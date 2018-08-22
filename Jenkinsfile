#!groovy 
pipeline {
	agent {
		node {
			label 'linuxSecondary'
			//customWorkspace '/some/other/path'
		}
	}
    environment {
        REPO_NAME = 'scmDockerDemo'
		LOD = '1.0.0'
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        SVNURL = 'http://svn.maximus.com/svn/'
        JAVA_OPTS='-Xmx1024m -Xms512m'
		installationName='http://cocd1mmweb01scm.maxcorp.maximus:9000'
    }
    options {
        buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '10', numToKeepStr: '20'))
        timestamps()
        retry(1)
        timeout time:10, unit:'MINUTES'
    }
	parameters {
        string(defaultValue: "dev_1.0.0", description: 'Branch Specifier', name: 'SPECIFIER')
        booleanParam(defaultValue: false, description: 'Deploy to QA Environment ?', name: 'DEPLOY_QA')
        booleanParam(defaultValue: false, description: 'Deploy to UAT Environment ?', name: 'DEPLOY_UAT')
        booleanParam(defaultValue: false, description: 'Deploy to PROD Environment ?', name: 'DEPLOY_PROD')
    }
    stages {
        stage("Initialize") {
            steps {
                script {
                    notifyBuild('STARTED')
                    echo "${BUILD_NUMBER} - ${env.BUILD_ID} on ${env.JENKINS_URL}"
                    echo "Branch Specifier :: ${params.SPECIFIER}"
                    echo "Deploy to QA? :: ${params.DEPLOY_QA}"
                    echo "Deploy to UAT? :: ${params.DEPLOY_UAT}"
                    echo "Deploy to PROD? :: ${params.DEPLOY_PROD}"
                }
            }
        }
        stage('Checkout') {
            steps {
                echo 'Checkout Repo'
                checkout([$class: 'SubversionSCM', additionalCredentials: [], browser: [$class: 'CollabNetSVN', url: 'http://svn.maximus.com/viewvc/scmDockerDemo/'], excludedCommitMessages: '', excludedRegions: '/snapshot/*', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[credentialsId: '', depthOption: 'infinity', ignoreExternalsOption: true, local: '.', remote: 'http://svn.maximus.com/svn/scmDockerDemo/lods/dev_1.0.0']], workspaceUpdater: [$class: 'CheckoutUpdater']])
            }
        }
        stage('Build') {
            steps {
				sh 'gradle --version'
            }
        }
        stage('Static Code Coverage Analysis') {
            parallel {
              stage('Execute Dependency Analysis') {
                  steps {
					  echo 'Dependency Scanning'
                      //dependencyCheckAnalyzer outdir: build/reports, datadir: depdata, suppressionFile: false, hintsFile: false, zipExtensions: false, scanpath: libs, includeCsvReports: false, includeHtmlReports: false, includeJsonReports: false, includeVulnReports: false, isAutoupdateDisabled: false, skipOnScmChange: false, skipOnUpstreamChange: false
                  }
              }
              stage('SonarQube analysis') {
                  steps {
						echo 'Sonarqube Scanning'
					//  withSonarQubeEnv {
					//	gradle sonarqube
					//  }
                  }
              }
            }
        }
        stage('Docker Build, Version & Publish') {
			steps {
				node('Docker') {
					 sh 'cd /Jenkins/scmScript;java -Xmx512m -Xms256m -jar scmScript.jar command CreateRelease -p ${REPONAME} -l dev_${LOD}'
                }
			 } 
		 }
        stage('Deploy - CI') {
            steps {
                echo "Deploying to CI Environment."
				input id: 'UDUCP', message: 'Update scmdockerdemo service?', parameters: [[$class: 'DynamicReferenceParameter', choiceType: 'ET_TEXT_BOX', description: '', name: 'SNAPSHOT', omitValueField: false, randomName: 'choice-parameter-18754605303716994', referencedParameters: 'PROJECT', script: [$class: 'ScriptlerScript', parameters: [PROJECT: 'scmDockerDemo', '': ''], scriptlerScriptId: 'SNAPSHOT.groovy']]]
            }
        }

        stage('Deploy - QA') {
            when {
                expression {
                    params.DEPLOY_QA == true
                }
            }
            steps {
                echo "Deploy to QA..."
            }
        }
        stage('Deploy - UAT') {
            when {
                expression {
                    params.DEPLOY_UAT == true
                }
            }
            steps {
                echo "Deploy to UAT..."
            }
        }
        stage('Deploy - Production') {
            when {
                expression {
                    params.DEPLOY_PROD == true
                }
            }
            steps {
                echo "Deploy to PROD..."
            }
        }
    }

    post {
        /*
         * These steps will run at the end of the pipeline based on the condition.
         * Post conditions run in order regardless of their place in pipeline
         * 1. always - always run
         * 2. changed - run if something changed from last run
         * 3. aborted, success, unstable or failure - depending on status
         */
        always {
            echo "I AM ALWAYS first"
            notifyBuild("${currentBuild.currentResult}")
        }
        aborted {
            echo "BUILD ABORTED"
        }
        success {
            echo "BUILD SUCCESS"
            echo "Keep Current Build If branch is master"
//            keepThisBuild()
        }
        unstable {
            echo "BUILD UNSTABLE"
        }
        failure {
            echo "BUILD FAILURE"
        }
    }
}
def getCurrentSnapshot() {
    return sh(returnStdout: true, script: "/usr/bin/java -Xmx512m -Xms256m -cp /opt/cie/bin/scmScript.jar:/opt/cie/bin/lib/*.jar com.maximus.scm.GetLatestSnapshot -r scmDockerDemo -l dev_1.0.0").trim()
}

def keepThisBuild() {
    currentBuild.setKeepLog(true)
    currentBuild.setDescription("Ship it! It is a hit")
}

def getShortCommitHash() {
    return sh(returnStdout: true, script: "echo get commit log").trim()
}

def getChangeAuthorName() {
    return sh(returnStdout: true, script: "echo get auth").trim()
}

def getChangeAuthorEmail() {
    return sh(returnStdout: true, script: "echo auth email").trim()
}

def getChangeSet() {
    return sh(returnStdout: true, script: 'echo changes').trim()
}

def getChangeLog() {
    return sh(returnStdout: true, script: "echo change log").trim()
}

def getCurrentBranch () {
    return sh (
            script: 'echo dev_1.0.0',
            returnStdout: true
    ).trim()
}

def isPRMergeBuild() {
    return (env.BRANCH_NAME ==~ /^PR-\d+$/)
}

def notifyBuild(String buildStatus = 'STARTED') {
    // build status of null means successful
    buildStatus = buildStatus ?: 'SUCCESS'

    def branchName = getCurrentBranch()
    def shortCommitHash = getShortCommitHash()
    def changeAuthorName = getChangeAuthorName()
    def changeAuthorEmail = getChangeAuthorEmail()
    def changeSet = getChangeSet()
    def changeLog = getChangeLog()

    // Default values
    def colorName = 'RED'
    def colorCode = '#FF0000'
    def subject = "${buildStatus}: '${env.JOB_NAME} [${env.BUILD_NUMBER}]'" + branchName + ", " + shortCommitHash
    def summary = "Started: Name:: ${env.JOB_NAME} \n " +
            "Build Number: ${env.BUILD_NUMBER} \n " +
            "Build URL: ${env.BUILD_URL} \n " +
            "Short Commit Hash: " + shortCommitHash + " \n " +
            "Branch Name: " + branchName + " \n " +
            "Change Author: " + changeAuthorName + " \n " +
            "Change Author Email: " + changeAuthorEmail + " \n " +
            "Change Set: " + changeSet

    if (buildStatus == 'STARTED') {
        color = 'YELLOW'
        colorCode = '#FFFF00'
    } else if (buildStatus == 'SUCCESS') {
        color = 'GREEN'
        colorCode = '#00FF00'
    } else {
        color = 'RED'
        colorCode = '#FF0000'
    }

    // Send notifications
	slackSend (baseUrl: 'https://maximusworkspace.slack.com/services/hooks/jenkins-ci/', 
	channel: 'scmdockerdemo', 
	color: color, 
	message: '${currentSNAPSHOT} is ' + summary, 
	tokenCredentialId: 'jenkins-slack-integration')

    if (buildStatus == 'FAILURE') {
        emailext attachLog: true, body: summary, compressLog: true, recipientProviders: [brokenTestsSuspects(), brokenBuildSuspects(), culprits()], replyTo: 'noreply@yourdomain.com', subject: subject, to: 'mpatel@yourdomain.com'
    }
}
