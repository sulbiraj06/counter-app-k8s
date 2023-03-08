pipeline{
    tools {
        maven 'maven3'
    }
    agent any

    parameters {
        choice(name : 'action', choices: 'create\ndestroy', description : 'Create or Destroy the eks cluster') 
        string(name : 'cluster', defaultValue : 'demo-eks', description : 'EKS cluster name') 
        string(name : 'region', defaultValue : 'us-east-1', description : 'EKS cluster region')
    }

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
        stage('EKS connect') { 
            steps { 
                sh """
                    aws eks --region ${params.region} update-kubeconfig --name ${params.cluster} 
                """;
            }
        }
        stage('EKS deployments') { 
            when { expression { params.action == 'create'}} 
            steps {
                script { 
                    def apply = false 
                    try{
                        input message : 'please confirm to apply the deploymnts', ok : 'Ready to apply' 
                        apply = true
                    }
                    catch(err) {
                        apply = false
                        CurrentBuild.result= 'UNSTABLE'
                    }
                    if(apply) {
                        sh """
                            kubectl apply -f . 
                        """;
                    }
                }
            }
        }
        stage('EKS Destroy') { 
            when { expression { params.action == 'destroy'}} 
            steps {
                script { 
                    def destroy = false 
                    try{
                        input message : 'please confirm to destroy the deploymnts', ok : 'Ready to destroy' 
                        destroy = true
                    }
                    catch(err) {
                        destroy = false
                        CurrentBuild.result= 'UNSTABLE'
                    }
                    if(destroy) {
                        sh """
                            kubectl delete -f . 
                        """;
                    }
                }
            }
        }
    }

}