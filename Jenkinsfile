pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
        jdk 'JDK11'
    }

    parameters {
        choice(name: 'Action', choices: ['Build', 'Deploy'], description: 'Build or Deploy')
    }

    environment {
        TIMESTAMP = new Date().format("yyyyMMdd.HHmmss")
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
                    def pomContent = readFile('pom.xml')
                    def versionMatch = pomContent =~ '<version>(.+?)</version>'
                    if (!versionMatch) {
                        error("Could not find <version> in pom.xml")
                    }

                    def baseVersion = versionMatch[0][1].trim()
                    echo "Base version from pom.xml: ${baseVersion}"

                    def finalVersion = "${baseVersion}-SNAPSHOT-dev-${env.BUILD_NUMBER}-${env.TIMESTAMP}"
                    echo "Final dev version: ${finalVersion}"

                    // Update pom.xml for build (not committed to Git)
                    bat "mvn versions:set -DnewVersion=${finalVersion}"
                    bat "mvn versions:commit"

                    env.VERSION = finalVersion
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
                echo "Simulated deployment. Artifacts:"
                bat 'dir target\\*.jar || echo No JAR found.'
            }
        }
    }

    post {
        always {
            echo "Final Version used for build: ${env.VERSION}"
        }
    }
}
