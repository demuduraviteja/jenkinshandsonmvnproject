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

                    // Skip <parent><version> and match the first <version> after <artifactId> in project scope
                    def matcher = pomContent =~ /<artifactId>[^<]+<\/artifactId>\s*<version>([^<]+)<\/version>/
                    def baseVersion = matcher ? matcher[0][1] : null

                    if (!baseVersion) {
                        error("Could not find <version> in pom.xml after artifactId")
                    }

                    echo "Base version from pom.xml: ${baseVersion}"

                    def finalVersion = "${baseVersion}-SNAPSHOT-dev-${env.BUILD_NUMBER}-${env.TIMESTAMP}"
                    echo "Final dev version: ${finalVersion}"

                    bat "mvn versions:set -DnewVersion=${finalVersion}"
                    bat "mvn versions:commit"

                    currentBuild.displayName = finalVersion
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
