pipeline {
    agent any

    tools {
        jdk 'jdk 17'
        maven 'maven3'
    }

    environment {
        SACNNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('git_check') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Kaleab-Walelign/Boardgame.git'
            }
        }
        stage('compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('test') {
            steps {
                sh "mvn test"
            }
        }
        stage('file system scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        stage('sonarqube analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SACNNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=.'''
                }
            }
        }
        stage('quality gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('build') {
            steps {
                sh "mvn package"
            }
        }
        stage('publish to nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk 17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -X"
                }
            }
        }
        stage('build and tag docker image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t kaleab00/boardshake:latest ."
                    }
                }
            }
        }
        stage('docker image scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html kaleab00/boardshake:latest"
            }
        }
        stage('push docker image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push kaleab00/boardshake:latest"
                    }
                }
            }
        }
        stage('deploy to k8s') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'default', contextName: '', credentialsId: 'kube-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.85.215:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('check deployment on k8s') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'default', contextName: '', credentialsId: 'kube-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.85.215:6443') {
                    sh "kubectl get pods"
                    sh "kubectl get svc"
                }
            }
        }
    }
}
