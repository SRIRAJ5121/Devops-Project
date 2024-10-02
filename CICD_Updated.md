## Install Plugins in Jenkins

1. **Eclipse Temurin Installer**:
   - This plugin enables Jenkins to automatically install and configure the Eclipse Temurin JDK (formerly known as AdoptOpenJDK).
   - To install, go to Jenkins dashboard -> Manage Jenkins -> Manage Plugins -> Available tab.
   - Search for "Eclipse Temurin Installer" and select it.
   - Click on the "Install without restart" button.

2. **Pipeline Maven Integration**:
   - This plugin provides Maven support for Jenkins Pipeline.
   - It allows you to use Maven commands directly within your Jenkins Pipeline scripts.
   - To install, follow the same steps as above, but search for "Pipeline Maven Integration" instead.

3. **Config File Provider**:
   - This plugin allows you to define configuration files (e.g., properties, XML, JSON) centrally in Jenkins.
   - These configurations can then be referenced and used by your Jenkins jobs.
   - Install it using the same procedure as mentioned earlier.

4. **SonarQube Scanner**:
   - SonarQube is a code quality and security analysis tool.
   - This plugin integrates Jenkins with SonarQube by providing a scanner that analyzes code during builds.
   - You can install it from the Jenkins plugin manager as described above.

5. **Docker**:
   - This plugin allows Jenkins to interact with Docker, enabling Docker builds and integration with Docker registries.
   - You can use it to build Docker images, run Docker containers, and push/pull images from Docker registries.
   - Install it from the plugin manager.

6. **Docker Pipeline Step**:
   - This plugin extends Jenkins Pipeline with steps to build, publish, and run Docker containers as part of your Pipeline scripts.
   - It provides a convenient way to manage Docker containers directly from Jenkins Pipelines.
   - Install it through the plugin manager like the others.

After installing these plugins, you may need to configure them according to your specific environment and requirements. This typically involves setting up credentials, configuring paths, and specifying options in Jenkins global configuration or individual job configurations. Each plugin usually comes with its own set of documentation to guide you through the configuration process.

## Configure Above Plugins in Jenkins as per video

## Pipeline 

```groovy

pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
    }
    
    stages {
        stage('GIT CHECKOUT') {
            steps {
               git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/VM2322/Boardgame.git'
            }
        }
    
                stage('MAVEN COMPILE') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('MAVEN TEST') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('TRIVY FILESYSTEM SCAN') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SONARQUBE ANALYSIS') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Gaming -Dsonar.projectKey=Gaming\
                            -Dsonar.java.binaries=. '''
                }
            }
        }
        
                stage('QUALITY GATE') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
        
        stage('MAVEN BUILD') {
            steps {
               sh "mvn package"
            }
        }
            stage('PUBLISH IMAGE TO NEXUS') {
            steps {
               withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        
       stage('BUILD & TAG DOCKER IMAGE') {
            steps {
               script {
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t VM2322/gaming:latest ."
                    }
               }
            }
        } 
    
             stage('IMAGE SCANNING') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html adijaiswal/boardshack:latest "
            }
        }   
            
        stage('DOCKER DEPLOYMENT') {
            steps {
                sh "docker run -d -p 8090:8080 VM2322/gaming:latest"
            }
        }   
    }
}
