pipeline {
    agent any
    tools {
        jdk 'Jdk17'
        maven 'Maven3'
    }
    environment {
        SCANNER_HOME = tool 'Sonar-Scanner-Tool'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', changelog: false, poll: false, url: 'https://github.com/ShubhamBhavsar101/Shopping-Cart.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn clean compile -DskipTests=true"
            }
        }
        stage('OWASP Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ ', odcInstallation: 'DP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Sonarqube') {
            steps {
                withSonarQubeEnv('Sonar-Localhost-Server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Shopping-Cart \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=Shopping-Cart '''
                }
            }
        }
        stage('Build') {
            steps {
                sh "mvn clean package -DskipTests=true"
            }
        }
        stage('Docker Build and Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'Docker') {
                        sh "docker build -t shopping-cart:latest -f docker/Dockerfile ."
                        sh "docker tag shopping-cart sneakyshubham/shopping-cart:latest"
                        sh "docker push sneakyshubham/shopping-cart:latest"
                    }
                }
            }
        }
        
    }
}
