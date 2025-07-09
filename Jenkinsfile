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
        choice(name: 'RELEVER', choices: ['major', 'minor', 'hotfix'], description: 'Release level: major, minor, hotfix')
    }

    environment {
        TIMESTAMP            = "${new Date().format('yyyyMMdd.HHmmss')}"
        GIT_REPO             = 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git'
        GIT_CREDENTIALS_ID   = 'github-PAT'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git(
                    branch: "${params.BRANCH_NAME}",
                    url: "${GIT_REPO}",
                    credentialsId: "${GIT_CREDENTIALS_ID}"
                )
            }
        }

        stage('Initialize Tools') {
            steps {
                echo "🔧 Checking tool versions"
                bat 'mvn --version'
                //bat 'java -version'
            }
        }

        stage('Validate Branch and Environment') {
            steps {
                script {
                    def isValid = (params.BRANCH_NAME.startsWith('master') || params.BRANCH_NAME.startsWith('hotfix')) && ENVIRONMENT == 'PROD'
                    if (!isValid) {
                        error "❌ Invalid combination: Branch = ${params.BRANCH_NAME}, Environment = ${ENVIRONMENT}"
                    }
                    echo "✅ Valid branch and environment"
                }
            }
        }

        stage('Parse Version and Determine Release') {
            steps {
                script {
                    // Run build-helper plugin to make parsedVersion available
                    bat 'mvn build-helper:parse-version'

                    if (params.RELEVER == 'major') {
                        def nextMajor = bat(script: "mvn help:evaluate -Dexpression=parsedVersion.nextMajorVersion -q -DforceStdout", returnStdout: true).trim()
                        RELEASE_VERSION = "${nextMajor}.0.0"
                    } else if (params.RELEVER == 'minor') {
                        def major = bat(script: "mvn help:evaluate -Dexpression=parsedVersion.majorVersion -q -DforceStdout", returnStdout: true).trim()
                        def nextMinor = bat(script: "mvn help:evaluate -Dexpression=parsedVersion.nextMinorVersion -q -DforceStdout", returnStdout: true).trim()
                        RELEASE_VERSION = "${major}.${nextMinor}.0"
                    } else if (params.RELEVER == 'hotfix') {
                        def major = bat(script: "mvn help:evaluate -Dexpression=parsedVersion.majorVersion -q -DforceStdout", returnStdout: true).trim()
                        def minor = bat(script: "mvn help:evaluate -Dexpression=parsedVersion.minorVersion -q -DforceStdout", returnStdout: true).trim()
                        def nextPatch = bat(script: "mvn help:evaluate -Dexpression=parsedVersion.nextIncrementalVersion -q -DforceStdout", returnStdout: true).trim()
                        RELEASE_VERSION = "${major}.${minor}.${nextPatch}"
                    } else {
                        error "❌ Invalid RELEVER: ${params.RELEVER}"
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
                        lines = readFile(filePath).split('\n').collect { it.trim() }.findAll { it }
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
            archiveArtifacts artifacts: "*/target/*.jar, */target/*.tar.gz", allowEmptyArchive: true
        }
        success {
            echo "✅ Build succeeded: ${FINAL_VERSION}"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
