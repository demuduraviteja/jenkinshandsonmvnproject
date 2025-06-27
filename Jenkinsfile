pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
        jdk 'JDK11'
    }

    parameters {
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git', description: 'Git repository URL')
        string(name: 'BRANCH_NAME', defaultValue: 'develop', description: 'Git branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'sit'], description: 'Target environment')
        choice(name: 'Action', choices: ['Build', 'Deploy'], description: 'Build or Deploy')
    }

    environment {
        TIMESTAMP = new Date().format("yyyyMMdd.HHmmss")
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "📥 Checking out branch: ${params.BRANCH_NAME}"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${params.BRANCH_NAME}"]],
                    userRemoteConfigs: [[url: "${params.GIT_REPO_URL}"]]
                ])
            }
        }

        stage('Initialize') {
            steps {
                echo "🔧 Verifying tools"
                bat 'mvn --version'
                bat 'java -version'
            }
        }

        stage('Validate Branch and Environment') {
            steps {
                script {
                    def isValid = false

                    if ((params.BRANCH_NAME == 'develop' || params.BRANCH_NAME.startsWith('develop')) && params.ENVIRONMENT == 'sit') {
                        isValid = true
                    } else if ((params.BRANCH_NAME.startsWith('feature') || params.BRANCH_NAME.startsWith('hotfix')) && params.ENVIRONMENT == 'dev') {
                        isValid = true
                    }

                    if (!isValid) {
                        error "❌ Invalid combination: Branch = ${params.BRANCH_NAME}, Environment = ${params.ENVIRONMENT}"
                    } else {
                        echo "✅ Valid combination: ${params.BRANCH_NAME} → ${params.ENVIRONMENT}"
                    }
                }
            }
        }

        stage('Auto-Increment Version') {
            steps {
                script {
                    def pomContent = readFile('pom.xml')
                    def versionMatch = (pomContent =~ /<version>([\d\.]+)<\/version>/)
                    if (!versionMatch || versionMatch.size() == 0) {
                        error "❌ Version not found in pom.xml"
                    }
                    def currentVersion = versionMatch[0][1].trim()
                    echo "📄 Current version in pom.xml: ${currentVersion}"

                    def (major, minor, patch) = currentVersion.tokenize('.').collect { it as int }
                    def incrementType = ''

                    if (params.BRANCH_NAME.startsWith('develop')) {
                        minor += 1
                        patch = 0
                        incrementType = 'minor'
                    } else if (params.BRANCH_NAME.startsWith('feature')) {
                        minor += 1
                        patch = 0
                        incrementType = 'minor'
                    } else if (params.BRANCH_NAME.startsWith('hotfix')) {
                        patch += 1
                        incrementType = 'patch'
                    } else {
                        error "❌ Unsupported branch for versioning: ${params.BRANCH_NAME}"
                    }

                    def newVersion = "${major}.${minor}.${patch}"
                    echo "🔁 Auto-incremented (${incrementType}) version: ${newVersion}"

                    bat "mvn versions:set -DnewVersion=${newVersion}"
                    bat "mvn versions:commit"

                    currentBuild.displayName = newVersion
                    writeFile file: 'build_version.txt', text: newVersion
                }
            }
        }

        stage('Build') {
            when {
                expression { params.Action == 'Build' }
            }
            steps {
                echo "🛠 Running Maven build"
                bat 'mvn clean install -DskipTests=true'
            }
        }

        stage('Deploy (Simulated)') {
            when {
                expression { params.Action == 'Deploy' }
            }
            steps {
                echo "🚀 Simulated deploy to ${params.ENVIRONMENT}"
                bat 'dir target\\*.jar || echo No JAR file found.'
            }
        }
    }

    post {
        success {
            script {
                def builtVersion = fileExists('build_version.txt') ? readFile('build_version.txt').trim() : 'Unknown'
                echo "✅ Build succeeded with version: ${builtVersion}"
            }
        }
        failure {
            echo "❌ Pipeline failed. Check the logs for errors."
        }
    }
}
