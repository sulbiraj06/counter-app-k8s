pipeline{
    tools {
        maven 'maven3'
    }
    agent any 
    stages{
        stage('Git Checkout'){
            steps{
                script{                 
                    git branch: 'master', url: 'https://github.com/sulbiraj06/counter-app-k8s.git'
                }
            }
        }
        stage('Unit Test'){
            steps{
                script{                   
                   sh 'mvn test'
                }
            }
        }
        stage('Integration Test'){
            steps{
                script{
                   sh 'mvn verify -DskipUnitTests'
                }
            }
        }
        stage('Maven Build'){
            steps{
                script{
                   sh 'mvn clean install'
                }
            }
        }
        stage('Static Code Analysis'){
            steps{
                script{
                  withSonarQubeEnv(credentialsId: 'sonar-api-key') { 
                    sh 'mvn clean package sonar:sonar'
                  }
                }
            }
        }
        stage('Quality Gate status Check'){
             steps{
                script{
                   waitForQualityGate abortPipeline: false, credentialsId: 'sonar-api-key'
                }
            }
        }
        stage('Upload JAR to Nexus'){
            steps{
                script{
                    def readPomVersion = readMavenPom file : 'pom.xml'
                    def nexusRepo = readPomVersion.version.endsWith("SNAPSHOT") ? "counterapp-snapshot" : "counterapp-release"
                    
                    nexusArtifactUploader artifacts: [[artifactId: 'springboot', classifier: '', file: 'target/Uber.jar', type: 'jar']], 
                    credentialsId: 'nexus-creds', 
                    groupId: 'com.example', 
                    nexusUrl: '3.213.160.126:8081', 
                    nexusVersion: 'nexus3', 
                    protocol: 'http', 
                    repository: nexusRepo, 
                    version: "${readPomVersion.version}"
                }
            }
        }
        stage('Build Docker image'){
             steps{
                script{
                   sh 'docker image build -t $JOB_NAME:v1.$BUILD_ID .'
                   sh 'docker image tag $JOB_NAME:v1.$BUILD_ID sulbiraj/$JOB_NAME:v1.$BUILD_ID'
                   sh 'docker image tag $JOB_NAME:v1.$BUILD_ID sulbiraj/$JOB_NAME:latest'
                }
            }
        }
        stage('Docker image push'){
             steps{
                script{
                   withCredentials([string(credentialsId: 'dockerhubpass', variable: 'dockerhubpasswd')]) {                     
                     sh 'docker login -u sulbiraj -p ${dockerhubpasswd}'
                     sh 'docker image push sulbiraj/$JOB_NAME:v1.$BUILD_ID'
                     sh 'docker image push sulbiraj/$JOB_NAME:latest'
                  }
                }
            }
        } 
    }
}