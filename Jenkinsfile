pipeline {
    
    agent any
	tools {
        maven "MAVEN3.9"
    }
    environment {
        registry = "ntaddese/vproappdock"
        registryCredential = 'dockerhub'
    }

    stages{

        stage('BUILD'){
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
        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }
        stage('INTEGRATION TEST'){
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
                def dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                env.dockerImage = dockerImage
              }
            }
        }
        stage('Upload Docker Image') {
            steps {
                script {
                        docker.withRegistry( '', registryCredential ) {
                            dockerImage.push("V$BUILD_NUMBER")
                            dockerImage.push('latest')
                    } 
                }
            }
        }
        /*
        stage('Remove Unused Docker Image') {
          steps{
            sh "docker rmi $registry:V$BUILD_NUMBER"
          }
        }
        */
        stage('Kubernetes Deploy') {
	       agent { label 'Helm' }
                steps {
                        sh "helm upgrade --install --force vproifle-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}