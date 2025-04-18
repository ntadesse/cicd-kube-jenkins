def COLOR_MAP = [
  'SUCCESS': 'good',
  'FAILURE': 'danger',
  ]
pipeline {
    
    agent any
	tools {
        maven "MAVEN3.9"
        jdk "JDK17"
        nodejs "NodeJS18"
    }
    environment {
        registry = "ntaddese/vproappdock"
        registryCredential = 'dockerhub'
        KUBECONFIG = credentials('kubeconfig')
    }

    stages{

        stage('CODE BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }
        /*
        stage('OWASP Dependency Check') {
                steps {
                    dependencyCheck additionalArguments: '''
                       --scan \'./target\'
                       --out \'./\'
                       --format \'ALL\' \
                       --prettyPrint''', odcInstallation: 'OWASP-DepCheck-10'
                    dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml', stopBuild: true
                 }
                 post {
                    always {
                        archiveArtifacts artifacts: 'dependency-check-report.*', allowEmptyArchive: true
                }
            }
       }
       */
        stage('CODE UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }
        stage('CODE INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile-kube \
                   -Dsonar.projectName=vprofile-kube \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.libraries=target/vprofile-v2/WEB-INF/lib/*.jar \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        } 
        stage('Building Docker Image') {
            steps{
              script {
                dockerImage = docker.build("${registry}:V${BUILD_NUMBER}")
              }
            }
        }
        stage ('Vulnerability Scan') {
            steps {
                sh '''
                trivy image "${registry}:V${BUILD_NUMBER}" \
                    --severity LOW,MEDIUM \
                    --exit-code 0 \
                    --quiet \
                    --format json -o trivy-image-MEDIUM-results.json
               
                trivy image "${registry}:V${BUILD_NUMBER}"  \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    --quiet \
                    --format json -o trivy-image-CRITICAL-results.json
                
            '''
            }
            post {
                always {
                    sh '''
                    trivy convert \
                        --format template  --template "@/usr/local/share/trivy/templates/html.tpl" \
                        --output trivy-image-MEDIUM-results.html trivy-image-MEDIUM-results.json
                    
                    trivy convert \
                        --format template  --template "@/usr/local/share/trivy/templates/html.tpl" \
                        --output trivy-image-CRITICAL-results.html trivy-image-CRITICAL-results.json
                    
                    trivy convert \
                        --format template  --template "@/usr/local/share/trivy/templates/junit.tpl" \
                        --output trivy-image-MEDIUM-results.xml trivy-image-MEDIUM-results.json
                    
                    trivy convert \
                        --format template  --template "@/usr/local/share/trivy/templates/junit.tpl" \
                        --output trivy-image-CRITICAL-results.xml trivy-image-CRITICAL-results.json
                
                    '''
                }
            }
        }
        stage('Upload Docker Image') {
            steps {
                script {
                        docker.withRegistry( '', registryCredential ) {
                           dockerImage.push("V${BUILD_NUMBER}")
                           dockerImage.push('latest')
                    } 
                }
            }
        }
        stage('Remove Unused Docker Image') {
          steps{
            sh "docker rmi ${registry}:V${BUILD_NUMBER}"
          }
        }
        stage('Kubernetes Deploy') {
	       agent { label 'Helm' }
                steps {
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh '''
                        export KUBECONFIG=/root/.kube/admin.conf
                        helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace app01-prod 
                        '''
                 } 
            }
        }
    }
    post {
        always {
        script {
            try {
                echo 'Slack Notifications.'
                slackSend channel: '#jenkinscicd',
                    color: COLOR_MAP[currentBuild.currentResult],
                    message: "*${currentBuild.currentResult}:*Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
            } catch (Exception e) {
                echo "Slack notification failed: ${e.getMessage()}"
            }
        }
            // JUnit reports
            // junit allowEmptyResults: true, testResults: '**/dependency-check-junit.xml'
            junit allowEmptyResults: true, skipMarkingBuildUnstable: true, testResults: '**/target/surefire-reports/*.xml'
            junit allowEmptyResults: true, skipMarkingBuildUnstable: true, testResults: '**/trivy-image-CRITICAL-results.xml'
            junit allowEmptyResults: true, skipMarkingBuildUnstable: true, testResults: '**/trivy-image-MEDIUM-results.xml'
            // Publish HTML reports
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'trivy-image-CRITICAL-results.html', reportName: 'Trivy Vulnerability Report (Critical)', reportTitles: '', useWrapperFileDirectly: true])
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'trivy-image-MEDIUM-results.html', reportName: 'Trivy Vulnerability Report (Medium)', reportTitles: '', useWrapperFileDirectly: true])
            // Archive all reports
            archiveArtifacts artifacts: '**/*.html, **/*.xml', allowEmptyArchive: true
       }
    }
}