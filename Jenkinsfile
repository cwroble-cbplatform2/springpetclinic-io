library 'cb-shared-lib@main'
def mvnPodYaml = libraryResource 'podtemplates/maven/pod.yml'
def kubectlPodYaml = libraryResource 'podtemplates/kubectl.yml'
environment {
    SONAR_CRED = credentials('thunder-sonar')
    controllerPodName = ''
}

pipeline {
  agent any
  options { 
    buildDiscarder(logRotator(numToKeepStr: '10'))
    skipDefaultCheckout true
    preserveStashes(buildCount: 10)
  }
  stages {
    stage('Extract Environment VAriables from Job') {
        steps {
          script {
            // Grab the pod name for the controller that kicks off the job
            env.controllerPodName = System.getenv('HOSTNAME')
            echo "Controller Pod Name extracted: ${env.controllerPodName}"
          }
        }
    }
    stage('Build & Scan') {
      agent{
        kubernetes {
          //inheritFrom 'maven-app'
          yaml mvnPodYaml
        }
      }
      stages{
        stage('Maven Build') {
          steps {
            checkout scm
            container('open-jdk17'){
              sh 'ls'
              sh 'java --version'
              sh 'echo $HOME'
              sh './mvnw clean package -Dcheckstyle.skip'
              sh "ls -l ${WORKSPACE}/target/"
              stash name: 'petclinic-jar', includes: 'target/spring-petclinic-3.3.0-SNAPSHOT.jar '
            }
          }  
        }
        stage('SonarQube Analysis') {
          steps {
            checkout scm
            container('open-jdk17'){
              withCredentials([string(credentialsId: 'thunder-sonar', variable: 'SONAR_SECRET')]) {
                sh "./mvnw sonar:sonar \
                -Dsonar.sourceEncoding=UTF-8 \
                -Dsonar.language=java \
                -Dsonar.projectKey=petclinic-1 \
                -Dsonar.host.url=https://sonarqube.cb-demos.io \
                -Dsonar.login=${SONAR_SECRET} \
                -Dsonar.projectName=petclinic-1 \
                -Dsonar.tests=src/test \
                -Dsonar.sources=src/main \
                -Dsonar.junit.reportsPath=target/surefire-reports \
                -Dsonar.surefire.reportsPath=target/surefire-reports \
                -Dsonar.jacoco.reportPath=target/jacoco.exec \
                -Dsonar.java.binaries=target/classes \
                -Dsonar.java.coveragePlugin=jacoco"
              }
            }
          }
        }
      }
    }
    
    stage('CheckMarx Results') {
        steps {
            writeFile(
                file: "checkmarx.json",
                text: '''\
                [{
                    "TotalIssues": 6,
                    "HighIssues": 0,
                    "MediumIssues": 1,
                    "LowIssues": 5,
                    "InfoIssues": 0,
                    "SastIssues": 6,
                    "KicsIssues": 0,
                    "ScaIssues": -1,
                    "APISecurity": {
                        "api_count": 0,
                        "total_risks_count": 0,
                        "risks": null
                    },
                    "RiskStyle": "medium",
                    "RiskMsg": "Medium Risk",
                    "Status": "Completed",
                    "ScanID": "8a072853-9594-47e5-a544-d6b2d4af837c",
                    "ScanDate": "",
                    "ScanTime": "",
                    "CreatedAt": "2023-07-06, 13:04:53",
                    "ProjectID": "e34d40c7-cd4b-4cba-9794-0004f66a173e",
                    "BaseURI": "https://ast.checkmarx.net/projects/e34d40c7-cd4b-4cba-9794-0004f66a173e/overview",
                    "Tags": {},
                    "ProjectName": "bws_enterpriseservices",
                    "BranchName": "testing-2",
                    "ScanInfoMessage": "",
                    "EnginesEnabled": [
                        "sast"
                    ]
                }]'''.stripIndent()
            )
        } // mock out CheckMarx results
    }
   
    stage('Publish') {
      agent any
        steps {
          unstash 'petclinic-jar'
          echo "Publish petclinic-jar to Nexus"
        }
    }

    stage('Trigger Release') {
      agent any
        steps {
          echo "Deploy petclinic-jar to GCP"       
        }
    }
  } //close stages
} //pipeline conclusion
