pipeline {
    agent any

    stages {
        stage('Checkout Java Application') {
            steps {
                // Clone the Java application repository
                git branch: 'vault-dev', 
                    url: 'https://gitlab.synesisit.info/convay/application/backend/file-service.git',
                    credentialsId: 'fileservice3'
            }
        }
        stage('Clone Configuration Repo') {
            steps {
                script {
                    // Clone the configuration repository
                    dir('/home/service/vault-fileservice-prop') {
                        git branch: 'main',
                            url: 'https://gitlab.synesisit.info/md.rasel/vault-fileservice-prop.git',
                            credentialsId: 'fileservice3'
                    }
                }
            }
        }
        stage('Copy Configuration Files') {
            steps {
                script {
                    // Copy application.properties and bootstrap.yml from the configuration repository
                    sh '''
                    cp /home/service/vault-fileservice-prop/application.properties src/main/resources/application.properties
                    cp /home/service/vault-fileservice-prop/bootstrap.yml src/main/resources/bootstrap.yml
                    '''
                }
            }
        }
        stage('Build') {
            agent {
                docker {
                    reuseNode true
                    image 'maven:3.8.4-jdk-8'
                    args '-v "maven-local:/root/.m2"'
                }
            }
            steps {
                // Run the Maven build
                sh 'mvn clean install -X'
            }
        }
        stage('Transfer JAR to Staged Directory') {
            steps {
                script {
                    // Transfer the built JAR file to the target directory (/home/services/staged/file-service)
                    sshPublisher(
                        continueOnError: false,
                        failOnError: true,
                        publishers: [
                            sshPublisherDesc(
                                configName: "chat@prod",  // Use the correct SSH configuration
                                transfers: [
                                    sshTransfer(
                                        sourceFiles: 'target/*.jar',
                                        removePrefix: 'target',
                                        remoteDirectory: "file-service",
                                        execCommand: "cd /home/services/staged/ && docker compose up -d --build file-service"
                                    ),
                                    sshTransfer(
                                        execCommand: '''
                                            cd /home/services/staged/file-service && \
                                            docker run --rm \
                                            -e SONAR_HOST_URL="http://172.16.28.31:9000" \
                                            -e SONAR_SCANNER_OPTS="-Dsonar.projectKey=fileservice-vault-dev" \
                                            -e SONAR_TOKEN="sqp_77ebb139e47a3f401936df7a6ad588dd94b599b1"  \
                                            -v "$(pwd):/usr/src"  sonarsource/sonar-scanner-cli && \
                                            docker compose build file-service && \
                                            docker compose up -d --force-recreate file-service
                                        '''
                                    )
                                    //sshTransfer(
                                    //    sourceFiles: 'target/*.jar',
                                    //    removePrefix: 'target',
                                    //    remoteDirectory: "/home/services/staged",
                                    //    execCommand: "echo 'JAR file transferred to /home/services/staged'"
                                    
                                ],
                                verbose: true
                            )
                        ]
                    )
                }
            }
        }
    }
    post {
        always {
            echo "Cleaning up workspace..."
            cleanWs()
        }
    }
}
