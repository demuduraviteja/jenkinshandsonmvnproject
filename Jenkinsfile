def FINAL_VERSION = '0'
def RELEASE_VERSION = ''
def SNAPSHOT_VERSION = ''
def ENVIRONMENT = 'PROD'

pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
        // jdk 'java-11-openjdk'
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'master', description: 'Git branch to build')
        choice(name: 'RELEVER', choices: ['major', 'minor', 'hotfix'], description: 'Release level')
    }

    environment {
        TIMESTAMP          = "${new Date().format('yyyyMMdd.HHmmss')}"
        GIT_REPO           = 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git'
        GIT_CREDENTIALS_ID = 'github-PAT'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: "${params.BRANCH_NAME}", url: "${GIT_REPO}", credentialsId: "${GIT_CREDENTIALS_ID}"
            }
        }

        stage('Initialize Tools') {
            steps {
                echo "🔧 Checking tool versions"
                bat 'mvn --version'
            }
        }

        stage('Validate Branch and Environment') {
            steps {
                script {
                    if (!(params.BRANCH_NAME.startsWith('master') || params.BRANCH_NAME.startsWith('hotfix')) || ENVIRONMENT != 'PROD') {
                        error "❌ Invalid combination: Branch = ${params.BRANCH_NAME}, ENV = ${ENVIRONMENT}"
                    }
                    echo "✅ Valid branch and environment"
                }
            }
        }

        stage('Parse Version and Determine Release') {
            steps {
                script {
                    bat 'mvn build-helper:parse-version'

                    def extractProp = { propName ->
                        def file = "tmp_${propName}.txt"
                        bat "del ${file} >nul 2>&1"
                        bat "mvn help:evaluate -Dexpression=${propName} -q -DforceStdout > ${file}"
                        return readFile(file).readLines().find { it.trim() && !it.contains("Downloading") && !it.contains("WARNING") }?.trim()
                    }

                    if (params.RELEVER == 'major') {
                        def nextMajor = extractProp('parsedVersion.nextMajorVersion')
                        if (!nextMajor) error "❌ Failed to extract nextMajorVersion"
                        RELEASE_VERSION = "${nextMajor}.0.0"

                    } else if (params.RELEVER == 'minor') {
                        def major = extractProp('parsedVersion.majorVersion')
                        def nextMinor = extractProp('parsedVersion.nextMinorVersion')
                        if (!major || !nextMinor) error "❌ Failed to extract minor version info"
                        RELEASE_VERSION = "${major}.${nextMinor}.0"

                    } else if (params.RELEVER == 'hotfix') {
                        def major = extractProp('parsedVersion.majorVersion')
                        def minor = extractProp('parsedVersion.minorVersion')
                        def patch = extractProp('parsedVersion.nextIncrementalVersion')
                        if (!major || !minor || !patch) error "❌ Failed to extract hotfix version info"
                        RELEASE_VERSION = "${major}.${minor}.${patch}"
                    }

                    SNAPSHOT_VERSION = "${RELEASE_VERSION}-SNAPSHOT"
                    echo "🏷️ Computed RELEASE_VERSION=${RELEASE_VERSION}, SNAPSHOT_VERSION=${SNAPSHOT_VERSION}"
                }
            }
        }

        stage('Maven Release') {
            steps {
                script {
                    bat """
                        mvn release:clean release:prepare release:perform -B ^
                        -DreleaseVersion=${RELEASE_VERSION} ^
                        -DdevelopmentVersion=${SNAPSHOT_VERSION} ^
                        -Dtag=release-${RELEASE_VERSION}
                    """
                }
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
                        lines = readFile(filePath).split('\n')*.trim().findAll()
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
                bat "mvn clean install -DskipTests=true"
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
