pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
        jdk 'JDK11'
    }

    parameters {
        choice(name: 'Action', choices: ['Build', 'Deploy'], description: 'Build or Deploy')
    }

    stages {
        stage('Initialize') {
            steps {
                bat 'mvn --version'
                bat 'java -version'
            }
        }

        stage('Determine Version') {
            steps {
                script {
                    def rawVersion = readMavenPom().getVersion()
                    def baseVersion = rawVersion
                    def timestamp = new Date().format("yyyyMMdd.HHmmss")
                    def fullVersion = "${baseVersion}-SNAPSHOT-dev-${BUILD_NUMBER}-${timestamp}"
                    env.VERSION = fullVersion

                    bat "mvn versions:set -DnewVersion=${env.VERSION}"
                    bat "mvn versions:commit"

                    echo "Generated Dev Version: ${env.VERSION}"
                }
            }
        }

        stage('Build') {
            when {
                expression { params.Action == 'Build' }
            }
            steps {
                bat 'mvn clean install -DskipTests=true'
            }
        }

        stage('Deploy (Simulated)') {
            when {
                expression { params.Action == 'Deploy' }
            }
            steps {
                echo "Simulating deploy... No actual push."
                bat 'dir target\\*.jar || echo No jar built.'
            }
        }
    }

    post {
        always {
            echo "Lower environment build complete. Final Version: ${env.VERSION}"
        }
    }
}
