def FINAL_VERSION = '0'
def RELEASE_VERSION = ''
def SNAPSHOT_VERSION = ''
def ENVIRONMENT = 'PROD'

pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
        jdk 'java-11-openjdk'
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'master', description: 'Git branch to build')
        choice(name: 'RELEVER', choices: ['major', 'minor', 'hotfix'], description: 'Release level (major/minor/hotfix)')
    }

    environment {
        TIMESTAMP          = "${new Date().format('yyyyMMdd.HHmmss')}"
        GIT_REPO           = 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git'
        GIT_CREDENTIALS_ID = 'github-PAT'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: "${params.BRANCH_NAME}", url: "${env.GIT_REPO}", credentialsId: "${env.GIT_CREDENTIALS_ID}"
            }
        }

        stage('Initialize Tools') {
            steps {
                echo "🔧 Checking Maven version"
                bat '"%MAVEN_HOME%\\bin\\mvn" --version'
            }
        }

        stage('Parse Version & Determine Release Version') {
            steps {
                script {
                    def extractProp = { propName ->
                        def file = "tmp_${propName}.txt"
                        bat "del ${file} >nul 2>&1"
                        bat "\"%MAVEN_HOME%\\bin\\mvn\" build-helper:parse-version help:evaluate -Dexpression=${propName} -q -DforceStdout > ${file}"
                        def lines = readFile(file).readLines().findAll { it?.trim() && !it.contains("Downloading") }
                        if (!lines) {
                            error "❌ Failed to extract property: ${propName}"
                        }
                        return lines[0].trim()
                    }

                    if (params.RELEVER == 'major') {
                        def nextMajor = extractProp('parsedVersion.nextMajorVersion')
                        RELEASE_VERSION = "${nextMajor}.0.0"
                    } else if (params.RELEVER == 'minor') {
                        def major = extractProp('parsedVersion.majorVersion')
                        def nextMinor = extractProp('parsedVersion.nextMinorVersion')
                        RELEASE_VERSION = "${major}.${nextMinor}.0"
                    } else if (params.RELEVER == 'hotfix') {
                        def major = extractProp('parsedVersion.majorVersion')
                        def minor = extractProp('parsedVersion.minorVersion')
                        def patch = extractProp('parsedVersion.nextIncrementalVersion')
                        RELEASE_VERSION = "${major}.${minor}.${patch}"
                    }

                    SNAPSHOT_VERSION = "${RELEASE_VERSION}-SNAPSHOT"
                    echo "🏷️ Computed RELEASE_VERSION=${RELEASE_VERSION}, SNAPSHOT_VERSION=${SNAPSHOT_VERSION}"
                }
            }
        }

        stage('Maven Release') {
            steps {
                bat """
                    "%MAVEN_HOME%\\bin\\mvn" release:clean release:prepare release:perform -B ^
                    -DreleaseVersion=${RELEASE_VERSION} ^
                    -DdevelopmentVersion=${SNAPSHOT_VERSION} ^
                    -Dtag=release-${RELEASE_VERSION}
                """
            }
        }

        stage('Store Version and Build Metadata') {
            steps {
                script {
                    FINAL_VERSION = RELEASE_VERSION
                    def filePath = 'releases_build_map.log'
                    def build = env.BUILD_NUMBER
                    def entry = "${FINAL_VERSION}:${build}"

                    def lines = []
                    if (fileExists(filePath)) {
                        def content = readFile(filePath)
                        lines = content.readLines().collect { it.trim() }.findAll { it }
                    }

                    lines << entry
                    lines = lines.unique().takeRight(2)
                    writeFile file: filePath, text: lines.join('\n')

                    echo "📁 Stored: ${FINAL_VERSION}:${build}"
                    currentBuild.displayName = FINAL_VERSION
                }
            }
        }

        stage('Build & Package') {
            steps {
                echo "🛠️ Running Maven build"
                bat '"%MAVEN_HOME%\\bin\\mvn" clean install -DskipTests=true'
            }
        }
    }

    post {
        always {
            echo "📦 Archiving Artifacts"
            archiveArtifacts artifacts: "**/target/*.jar", allowEmptyArchive: true
        }
        success {
            echo "✅ Build succeeded: ${FINAL_VERSION}"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
