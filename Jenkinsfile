def FINAL_VERSION = '0'

pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
        // jdk 'java-11-openjdk'  // Uncomment if needed
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'develop', description: 'Git branch to build')
        choice(name: 'RELEVER', choices: ['major', 'minor', 'hotfix'], description: 'Release type (only for master/hotfix)')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'sit', 'prod'], description: 'Target environment')
    }

    environment {
        GIT_REPO = 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git'
        TIMESTAMP = "${new Date().format('yyyyMMdd.HHmmss')}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                echo "📥 Checking out branch: ${params.BRANCH_NAME}"
                git branch: "${params.BRANCH_NAME}", url: "${GIT_REPO}"
            }
        }

        stage('Validate Branch and Environment') {
            steps {
                script {
                    def valid = false
                    def branch = params.BRANCH_NAME
                    def env = params.ENVIRONMENT

                    if ((branch.startsWith('master') || branch.startsWith('hotfix')) && env == 'prod') {
                        valid = true
                    } else if ((branch.startsWith('develop') || branch.startsWith('feature') || branch.startsWith('bugfix')) &&
                               (env == 'dev' || env == 'sit')) {
                        valid = true
                    }

                    if (!valid) {
                        error "❌ Invalid branch + environment combination: ${branch} + ${env}"
                    } else {
                        echo "✅ Valid combination: ${branch} → ${env}"
                    }
                }
            }
        }

        stage('Determine and Set Version') {
            steps {
                script {
                    sh '''
                        git config --global user.name "demuduraviteja"
                        git config --global user.email "shanmukha2342@gmail.com"
                    '''

                    if (params.BRANCH_NAME.startsWith('master') || params.BRANCH_NAME.startsWith('hotfix')) {
                        echo "📦 Releasing for Production"

                        sh 'mvn build-helper:parse-version'

                        def major = sh(script: "mvn help:evaluate -Dexpression=parsedVersion.majorVersion -q -DforceStdout", returnStdout: true).trim()
                        def minor = sh(script: "mvn help:evaluate -Dexpression=parsedVersion.minorVersion -q -DforceStdout", returnStdout: true).trim()
                        def patch = sh(script: "mvn help:evaluate -Dexpression=parsedVersion.incrementalVersion -q -DforceStdout", returnStdout: true).trim()

                        def releaseVersion = ""
                        def nextSnapshot = ""

                        if (params.RELEVER == 'major') {
                            releaseVersion = "${major.toInteger() + 1}.0.0"
                            nextSnapshot   = "${major.toInteger() + 1}.0.0-SNAPSHOT"
                        } else if (params.RELEVER == 'minor') {
                            releaseVersion = "${major}.${minor.toInteger() + 1}.0"
                            nextSnapshot   = "${major}.${minor.toInteger() + 1}.0-SNAPSHOT"
                        } else if (params.RELEVER == 'hotfix') {
                            releaseVersion = "${major}.${minor}.${patch.toInteger() + 1}"
                            nextSnapshot   = "${major}.${minor}.${patch.toInteger() + 1}-SNAPSHOT"
                        }

                        echo "🏷️ Release Version: ${releaseVersion}, Next Snapshot: ${nextSnapshot}"

                        sh """
                            mvn release:clean release:prepare release:perform \
                              -B \
                              -DreleaseVersion=${releaseVersion} \
                              -DdevelopmentVersion=${nextSnapshot}
                        """

                        def pom = readMavenPom file: 'pom.xml'
                        FINAL_VERSION = pom.version
                        currentBuild.displayName = FINAL_VERSION

                    } else {
                        echo "📦 Dev/SIT auto-versioning using suffix"

                        def suffix = (params.ENVIRONMENT == 'sit') 
                            ? "SNAPSHOT-${env.BUILD_NUMBER}-${env.TIMESTAMP}" 
                            : "SNAPSHOT-dev-${env.BUILD_NUMBER}-${env.TIMESTAMP}"

                        if (params.BRANCH_NAME.startsWith('feature') || params.BRANCH_NAME.startsWith('develop')) {
                            echo "📈 Bumping minor version"
                            sh """
                                mvn build-helper:parse-version versions:set \
                                    -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.nextMinorVersion}.0-${suffix} \
                                    versions:commit
                            """
                        } else if (params.BRANCH_NAME.startsWith('hotfix') || params.BRANCH_NAME.startsWith('bugfix')) {
                            echo "🔧 Bumping patch version"
                            sh """
                                mvn build-helper:parse-version versions:set \
                                    -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion}-${suffix} \
                                    versions:commit
                            """
                        }

                        def pom = readMavenPom file: 'pom.xml'
                        FINAL_VERSION = pom.version
                        currentBuild.displayName = FINAL_VERSION
                    }
                }
            }
        }

        stage('Build') {
            steps {
                echo "🛠️ Building with version: ${FINAL_VERSION}"
                sh 'mvn clean install -DskipTests=true'
            }
        }
    }

    post {
        success {
            echo "✅ Build completed successfully with version: ${FINAL_VERSION}"
        }
        failure {
            echo "❌ Build failed"
        }
    }
}
