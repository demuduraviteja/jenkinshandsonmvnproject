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

        stage('Initialize') {
            steps {
                sh 'mvn --version'
                sh 'java -version || echo Java not installed'
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
                    // Step 1: Run build-helper plugin (required to fill parsedVersion)
                    sh "mvn build-helper:parse-version"

                    // Step 2: Extract version components one by one
                    def major     = sh(script: "mvn help:evaluate -Dexpression=parsedVersion.majorVersion -q -DforceStdout", returnStdout: true).trim()
                    def minor     = sh(script: "mvn help:evaluate -Dexpression=parsedVersion.minorVersion -q -DforceStdout", returnStdout: true).trim()
                    def patch     = sh(script: "mvn help:evaluate -Dexpression=parsedVersion.incrementalVersion -q -DforceStdout", returnStdout: true).trim()
                    def nextMajor = sh(script: "mvn help:evaluate -Dexpression=parsedVersion.nextMajorVersion -q -DforceStdout", returnStdout: true).trim()
                    def nextMinor = sh(script: "mvn help:evaluate -Dexpression=parsedVersion.nextMinorVersion -q -DforceStdout", returnStdout: true).trim()
                    def nextPatch = sh(script: "mvn help:evaluate -Dexpression=parsedVersion.nextIncrementalVersion -q -DforceStdout", returnStdout: true).trim()

                    echo "Parsed: major=${major}, minor=${minor}, patch=${patch}, nextMajor=${nextMajor}, nextMinor=${nextMinor}, nextPatch=${nextPatch}"

                    if (params.RELEVER == 'major') {
                        RELEASE_VERSION = "${nextMajor}.0.0"
                    } else if (params.RELEVER == 'minor') {
                        RELEASE_VERSION = "${major}.${nextMinor}.0"
                    } else if (params.RELEVER == 'hotfix') {
                        RELEASE_VERSION = "${major}.${minor}.${nextPatch}"
                    }

                    SNAPSHOT_VERSION = "${RELEASE_VERSION}-SNAPSHOT"
                    echo "📊 Computed RELEASE_VERSION=${RELEASE_VERSION}, SNAPSHOT_VERSION=${SNAPSHOT_VERSION}"
                }
            }
        }

        stage('Maven Release') {
            steps {
                sh """
                    mvn release:clean release:prepare release:perform -B \\
                        -DreleaseVersion=${RELEASE_VERSION} \\
                        -DdevelopmentVersion=${SNAPSHOT_VERSION} \\
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
                        lines = readFile(filePath).readLines().collect { it.trim() }.findAll { it }
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
                sh "mvn clean install -DskipTests=true"
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
