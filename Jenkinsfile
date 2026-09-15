pipeline {
    agent {
        docker {
            image 'maven:3.9.9-eclipse-temurin-21'
            // 复用本机 Maven 缓存，加速二次构建
            args '-v $HOME/.m2:/root/.m2'
        }
    }

    options {
        timestamps()
        // 构建产物保留天数，避免磁盘被占满
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Deliver') {
            steps {
                sh 'chmod +x ./jenkins/scripts/deliver.sh && ./jenkins/scripts/deliver.sh'
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            echo 'Pipeline succeeded'
        }
        failure {
            echo 'Pipeline failed — check Console Output'
        }
        always {
            cleanWs()
        }
    }
}
