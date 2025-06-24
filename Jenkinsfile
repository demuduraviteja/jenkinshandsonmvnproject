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
                    // Immediately extract just the string to avoid passing regex object
                    def baseVersion = pomContent.find(/<version>(.+?)<\/version>/) { match, ver -> return ver }

                    if (!baseVersion) {
                        error("Could not find <version> in pom.xml")
                    }

                    echo "Base version from pom.xml: ${baseVersion}"

                    def finalVersion = "${baseVersion}-SNAPSHOT-dev-${env.BUILD_NUMBER}-${env.TIMESTAMP}"
                    echo "Final dev version: ${finalVersion}"

                    // Update pom.xml for build (not committed to Git)
                    bat "mvn versions:set -DnewVersion=${finalVersion}"
                    bat "mvn versions:commit"

                    // Assign version to environment in safe way
                    currentBuild.displayName = finalVersion
                    // Store in a file for later post section
                    writeFile file: 'build_version.txt', text: finalVersion
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
            script {
                def builtVersion = fileExists('build_version.txt') ? readFile('build_version.txt').trim() : 'Unknown'
                echo "Final Version used for build: ${builtVersion}"
            }
        }
    }
}
