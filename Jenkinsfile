pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
        jdk 'JDK11'
    }

    parameters {
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git', description: 'Git repository URL')
        string(name: 'BRANCH_NAME', defaultValue: 'develop', description: 'Git branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'sit'], description: 'Deployment Environment')
        choice(name: 'Action', choices: ['Build', 'Deploy'], description: 'Choose Build or Deploy')
    }

    environment {
        TIMESTAMP = new Date().format("yyyyMMdd.HHmmss")
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${params.BRANCH_NAME}"]],
                    userRemoteConfigs: [[url: "${params.GIT_REPO_URL}"]]
                ])
            }
        }

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
                    def lines = pomContent.split('\n')
                    def baseVersion = null
                    def inParent = false

                    for (line in lines) {
                        if (line.contains('<parent>')) inParent = true
                        if (line.contains('</parent>')) inParent = false

                        if (!inParent && line.trim() =~ /<version>(.+)<\/version>/) {
                            def matcher = (line.trim() =~ /<version>(.+)<\/version>/)
                            if (matcher) {
                                baseVersion = matcher[0][1].trim()
                                break
                            }
                        }
                    }

                    if (!baseVersion) {
                        error("Could not find <version> in pom.xml (outside <parent>)")
                    }

                    echo "Base version: ${baseVersion}"
                    def finalVersion = ''

                    if (params.ENVIRONMENT == 'sit' && params.BRANCH_NAME.startsWith('develop')) {
                        finalVersion = "${baseVersion}-SNAPSHOT-${env.BUILD_NUMBER}-${env.TIMESTAMP}-${env.BUILD_ID}"
                    } else if (params.ENVIRONMENT == 'dev' && (
                               params.BRANCH_NAME.startsWith('feature') ||
                               params.BRANCH_NAME.startsWith('hotfix') ||
                               params.BRANCH_NAME.startsWith('bugfix'))) {
                        finalVersion = "${baseVersion}-SNAPSHOT-dev-${env.BUILD_NUMBER}-${env.TIMESTAMP}-${env.BUILD_ID}"
                    } else {
                        error("Invalid branch/environment combination for versioning")
                    }

                    echo "Final Version: ${finalVersion}"

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
                echo "Simulated Deployment Output:"
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
