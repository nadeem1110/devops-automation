pipeline{
    agent any
    tools {
        maven 'maven'
    }
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "10.21.34.140:8081"
        NEXUS_REPOSITORY = "maven-pipeline"
        NEXUS_CREDENTIAL_ID = "NEXUS_CRED"
        tag = sh(returnStdout: true, script: "git rev-parse --short=10 HEAD").trim()
    }
    stages
    {
        stage('Build Maven') 
        {
        steps
            {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'Git-cred', url: 'https://github.com/nadeem1110/devops-automation.git']]])
                sh "mvn -Dmaven.test.failure.ignore=true clean package"
            }   
        }
        stage('Execute Sonarqube Report') 
        {
            steps 
            {
                withSonarQubeEnv('sonarqube') 
                {
                    sh "mvn sonar:sonar"
                }
            }
        }
        
        stage('Build Docker Image') 
        {
            steps 
            {
                script 
                {
                  sh 'docker build -t nadeempatel/devops-integration:${tag} .'
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                 withCredentials([string(credentialsId: 'Docker-cred', variable: 'dockerpass')]) {
                    sh 'docker login -u nadeempatel -p ${dockerpass}'
                 }  
                 sh 'docker push nadeempatel/devops-integration:${tag}'
                }
            }
        }
        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: "10.21.34.140:8081",
                            groupId: 'pom.org.springframework.boot',
                            version: '2.7.0',
                            repository: 'maven-pipeline',
                            credentialsId: 'NEXUS_CRED',
                            artifacts: [
                                [artifactId: 'pom.spring-boot-starter-parent',
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: 'pom.spring-boot-starter-parent',
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );
                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }
        stage('deploy to k8s')
        {
            steps
            {
                script
                {
                    kubernetesDeploy (configs: 'deploymentservice.yaml', kubeconfigId: 'k8sconfigpwd')
                }
            }
        }
        
    }
}
    
