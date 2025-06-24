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
                sh 'mvn --version'
                sh 'java -version'
            }
        }

        stage('Determine Version') {
            steps {
                script {
                    // Read <version> from pom.xml (e.g., 2.0.0)
                    def rawVersion = readMavenPom().getVersion()

                    // Use the raw version directly as base
                    def baseVersion = rawVersion

                    // Add suffixes for dev environment
                    def timestamp = new Date().format("yyyyMMdd.HHmmss")
                    def fullVersion = "${baseVersion}-SNAPSHOT-dev-${BUILD_NUMBER}-${timestamp}"

                    // Export version to environment
                    env.VERSION = fullVersion

                    // Update pom.xml with full version
                    sh "mvn versions:set -DnewVersion=${env.VERSION}"
                    sh "mvn versions:commit"

                    echo "Generated Dev Version: ${env.VERSION}"
                }
            }
        }

        stage('Build') {
            when {
                expression { params.Action == 'Build' }
            }
            steps {
                sh 'mvn clean install -DskipTests=true'
            }
        }

        stage('Deploy (Simulated)') {
            when {
                expression { params.Action == 'Deploy' }
            }
            steps {
                echo "Simulating deploy... No actual push."
                sh 'ls -l target/*.jar || echo "No jar built."'
            }
        }
    }

    post {
        always {
            echo "Lower environment build complete. Final Version: ${env.VERSION}"
        }
    }
}
