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
        TIMESTAMP = "${new Date().format('yyyyMMdd.HHmmss')}"
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
                    } else if ((params.BRANCH_NAME.startsWith('feature') || params.BRANCH_NAME.startsWith('hotfix') || params.BRANCH_NAME.startsWith('bugfix')) && params.ENVIRONMENT == 'dev') {
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
                    def versionLine = null
                    def insideParent = false

                    for (line in pomContent.readLines()) {
                        line = line.trim()
                        if (line.contains('<parent>')) {
                            insideParent = true
                        } else if (line.contains('</parent>')) {
                            insideParent = false
                        }

                        if (!insideParent && line.contains('<version>') && line.contains('</version>')) {
                            versionLine = line
                            break
                        }
                    }

                    if (!versionLine) {
                        error("❌ <version> tag not found in pom.xml (outside <parent>)")
                    }

                    def versionMatch = versionLine.replaceAll(/.*<version>([0-9]+)\.([0-9]+)\.([0-9]+)<\/version>.*/, '$1,$2,$3')
                    def (majorStr, minorStr, patchStr) = versionMatch.tokenize(',')

                    def major = majorStr.toInteger()
                    def minor = minorStr.toInteger()
                    def patch = patchStr.toInteger()

                    def incrementType = ''
                    if (params.BRANCH_NAME.startsWith('develop') || params.BRANCH_NAME == 'develop') {
                        minor += 1; patch = 0; incrementType = 'minor'
                    } else if (params.BRANCH_NAME.startsWith('feature')) {
                        minor += 1; patch = 0; incrementType = 'minor'
                    } else if (params.BRANCH_NAME.startsWith('hotfix') || params.BRANCH_NAME.startsWith('bugfix')) {
                        patch += 1; incrementType = 'patch'
                    } else {
                        error("❌ Unsupported branch for versioning: ${params.BRANCH_NAME}")
                    }

                    def autoVersion = "${major}.${minor}.${patch}"
                    echo "🔁 Auto-incremented (${incrementType}) version: ${autoVersion}"

                    def finalVersion = ''
                    if (params.ENVIRONMENT == 'sit') {
                        finalVersion = "${autoVersion}-SNAPSHOT-${env.BUILD_NUMBER}-${env.TIMESTAMP}-${env.BUILD_ID}"
                    } else {
                        finalVersion = "${autoVersion}-SNAPSHOT-dev-${env.BUILD_NUMBER}-${env.TIMESTAMP}-${env.BUILD_ID}"
                    }

                    echo "📦 Final Version to use: ${finalVersion}"

                    bat "mvn versions:set -DnewVersion=${finalVersion}"
                    bat "mvn versions:commit"

                    writeFile file: 'build_version.txt', text: finalVersion
                    currentBuild.displayName = finalVersion
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
