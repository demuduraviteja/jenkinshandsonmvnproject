pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
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
                git branch: 'master', credentialsId: "${GIT_CREDENTIALS_ID}", url: "${GIT_REPO}"
            }
        }

        stage('Verify Tools') {
            steps {
                sh 'mvn --version'
                sh 'git --version'
            }
        }

        stage('Parse Version from pom.xml') {
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    def currentVersion = pom.version.trim()

                    echo "📦 Current version from pom.xml = ${currentVersion}"

                    if (!currentVersion.endsWith("-SNAPSHOT")) {
                        error("❌ Current version must be a SNAPSHOT version, but got: ${currentVersion}")
                    }

                    def baseVersion = currentVersion.replace("-SNAPSHOT", "")
                    def parts = baseVersion.tokenize('.')
                    def major = parts[0] as int
                    def minor = parts[1] as int
                    def patch = parts[2] as int

                    // Just bump patch version for this example
                    def newReleaseVersion = "${major}.${minor}.${patch}"
                    def newSnapshotVersion = "${major}.${minor}.${patch + 1}-SNAPSHOT"
                    def newTag = "release-${newReleaseVersion}"

                    echo "✅ Release version = ${newReleaseVersion}"
                    echo "✅ Snapshot version = ${newSnapshotVersion}"
                    echo "🏷️ Git Tag = ${newTag}"

                    env.RELEASE_VERSION = newReleaseVersion
                    env.SNAPSHOT_VERSION = newSnapshotVersion
                    env.TAG_NAME = newTag
                }
            }
        }

        stage('Release with Tag') {
            steps {
                sh """
                    git config user.name "demuduraviteja"
                    git config user.email "shanmukha2342@gmail.com"

                    mvn release:clean release:prepare release:perform -B \\
                        -DreleaseVersion=${RELEASE_VERSION} \\
                        -DdevelopmentVersion=${SNAPSHOT_VERSION} \\
                        -Dtag=${TAG_NAME}
                """
            }
        }
    }

    post {
        success {
            echo "✅ Maven release complete with tag ${TAG_NAME}"
        }
        failure {
            echo "❌ Release failed. Check console for errors."
        }
    }
}
