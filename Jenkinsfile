pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
    }

    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: 'Git branch to build')
        choice(name: 'RELEVER', choices: ['major', 'minor', 'hotfix'], description: 'Version bump type')
    }

    environment {
        GIT_REPO           = 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git'
        GIT_CREDENTIALS_ID = 'github-PAT'
        RELEASE_VERSION    = ''
        SNAPSHOT_VERSION   = ''
        TAG_NAME           = ''
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: "${params.BRANCH}", credentialsId: "${GIT_CREDENTIALS_ID}", url: "${GIT_REPO}"
            }
        }

        stage('Parse Version from pom.xml') {
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    def currentVersion = pom.version.trim()

                    echo "📦 Current version from pom.xml: ${currentVersion}"

                    if (!currentVersion.endsWith("-SNAPSHOT")) {
                        error("❌ Version must end with '-SNAPSHOT'. Found: ${currentVersion}")
                    }

                    def baseVersion = currentVersion.replace("-SNAPSHOT", "")
                    def parts = baseVersion.tokenize('.')

                    if (parts.size() < 2) {
                        error("❌ Version must have at least major.minor (X.Y). Found: ${currentVersion}")
                    }

                    // Fill missing parts with 0
                    while (parts.size() < 3) {
                        parts << '0'
                    }

                    def major = parts[0] as int
                    def minor = parts[1] as int
                    def patch = parts[2] as int

                    // Auto bump based on RELEVER
                    if (params.RELEVER == 'major') {
                        major += 1
                        minor = 0
                        patch = 0
                    } else if (params.RELEVER == 'minor') {
                        minor += 1
                        patch = 0
                    } else if (params.RELEVER == 'hotfix') {
                        patch += 1
                    } else {
                        error("❌ Invalid RELEVER value: ${params.RELEVER}")
                    }

                    env.RELEASE_VERSION = "${major}.${minor}.${patch}"
                    env.SNAPSHOT_VERSION = "${major}.${minor}.${patch + 1}-SNAPSHOT"
                    env.TAG_NAME = "release-${env.RELEASE_VERSION}"

                    echo "✅ RELEASE_VERSION = ${env.RELEASE_VERSION}"
                    echo "🔄 SNAPSHOT_VERSION = ${env.SNAPSHOT_VERSION}"
                    echo "🏷️ TAG_NAME = ${env.TAG_NAME}"
                }
            }
        }

        stage('Run Maven Release') {
            steps {
                sh '''
                    git config user.name "demuduraviteja"
                    git config user.email "shanmukha2342@gmail.com"

                    mvn release:clean release:prepare release:perform -B \
                        -DreleaseVersion=${RELEASE_VERSION} \
                        -DdevelopmentVersion=${SNAPSHOT_VERSION} \
                        -Dtag=${TAG_NAME}
                '''
            }
        }

        stage('Build Verification') {
            steps {
                sh 'mvn clean install -DskipTests=true'
            }
        }
    }

    post {
        success {
            echo "✅ Build completed successfully with tag: ${TAG_NAME}"
        }
        failure {
            echo "❌ Build or release process failed."
        }
    }
}
