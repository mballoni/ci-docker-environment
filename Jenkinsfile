#!groovy

node {

    stage('Build') {
        echo "Running ${env.BUILD_ID} [branch:${env.BRANCH_NAME}] on ${env.JENKINS_URL}"
        deleteDir() //delete the cloned dir before each build
        checkout scm //Jenkins with multibranch support
        sh './gradlew clean build'
        stash includes: 'build/libs/*.jar', name: 'app-jar'
    }

    stage('Code Quality') {
        parallel publish_test_reports: {
            junit '**/build/test-results/test/*.xml' //publish test reports
        },
            sonar: {
                sh "./gradlew sonarqube"

                //qualityGate()
            },
            owasp_dependency_check: {
                sh './gradlew dependencyCheck --info'
            }
    }

    stage('Release') {
        sh './gradlew release -Prelease.useAutomaticVersion=true'
        sh './gradlew upload'
    }

}


/**
 * Call Sonar endpoints to extract/evaluate quality gate analysis result
 * @return
 */
def qualityGate() {
    def props = readProperties file: "build/sonar/report-task.txt"
    echo "Quality Gate Properties=${props}"
    def sonarServerUrl = props['serverUrl']
    def ceTaskUrl = props['ceTaskUrl']

    timeout(time: 3, unit: 'MINUTES') {
        waitUntil {
            ceTask = new URL(ceTaskUrl)
            def analysisStatus = getUrlJsonProperty(ceTaskUrl, "status")
            echo "Quality Gate analysisStatus: ${analysisStatus}"

            return "SUCCESS".equals(analysisStatus)
        }
    }
    def analysisId = getUrlJsonProperty(ceTaskUrl, "analysisId")
    echo "Quality Gate analysisId=${analysisId}"

    def qualigateUrl = sonarServerUrl + "/api/qualitygates/project_status?analysisId=" + analysisId
    def qualitygateStatus = getUrlJsonProperty(qualigateUrl, "status")
    echo "Quality Gate status=${qualitygateStatus}"
    if ("ERROR".equals(qualitygateStatus)) {
//        error "Sonar Quality Gate failure: ${qualitygateStatus}"
        currentBuild.result = 'UNSTABLE'
    }
}

/**
 * Make a request to url and extract a property value from json response
 * @param url
 * @param property
 * @return
 */
def getUrlJsonProperty(url, property) {
    sh(script: """
               curl -s -k ${url} | sed 's/{.*${property}":"*\\([0-9a-zA-Z]*\\)"*,*.*}/\\1/'
               """, returnStdout: true)
}

