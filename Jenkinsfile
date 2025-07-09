def FINAL_VERSION = '0'
def ENVIRONMENT = 'PROD'

pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
        //jdk 'java-11-openjdk'
    }

    environment {
        TIMESTAMP            = "${new Date().format('yyyyMMdd.HHmmss')}"
        GIT_REPO             = 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git'
        //NEXUS_REPO           = 'https://repo.td.com/repository/eets-staging-authenticated'
        //GROUP_ID             = 'BSM/LAMP'
        //ARTIFACT_ID          = 'slr-platform'
        GIT_CREDENTIALS_ID   = 'github-pat'
        //NEXUS_CREDENTIALS_ID = 'lamp_nexus'
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'develop', description: 'Git branch to build')
        choice(name: 'RELEVER', choices: ['major', 'minor', 'hotfix'], description: 'Release type')
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

        stage('Initialize') {
            steps {
                echo "🔧 Verifying tools"
                sh 'mvn --version'
                sh 'java -version'
            }
        }

        stage('Validate Branch and Environment') {
            steps {
                script {
                    def isValid = false
                    if (params.BRANCH_NAME.startsWith('master') && ENVIRONMENT == 'PROD') {
                        isValid = true
                    } else if (params.BRANCH_NAME.startsWith('hotfix') && ENVIRONMENT == 'PROD') {
                        isValid = true
                    }

                    if (!isValid) {
                        error "❌ Invalid combination: Branch = ${params.BRANCH_NAME}, Environment = ${ENVIRONMENT}"
                    } else {
                        echo "✅ Valid combination: ${params.BRANCH_NAME} - ${ENVIRONMENT}"
                    }
                }
            }
        }

        stage('Determine Release Version') {
            when {
                expression { params.BRANCH_NAME == 'master' && params.RELEVER == 'major' }
            }
            steps {
                script {
                    echo "🔍 Parsing current version and computing next major version..."

                    def nextMajor = sh(
                        script: """mvn help:evaluate -Dexpression=project.version -q -DforceStdout | sed 's/-SNAPSHOT//' | awk -F. '{print \$1 + 1}'""",
                        returnStdout: true
                    ).trim()

                    env.RELEASE_VERSION = "${nextMajor}.0.0"
                    env.NEXT_SNAPSHOT_VERSION = "${env.RELEASE_VERSION}-SNAPSHOT"

                    echo "📦 Calculated releaseVersion=${env.RELEASE_VERSION}, nextSnapshot=${env.NEXT_SNAPSHOT_VERSION}"
                }
            }
        }

        stage('Release Master with Tag') {
            when {
                expression { params.BRANCH_NAME == 'master' && params.RELEVER == 'major' }
            }
            steps {
                script {
                    echo "🏷️ Releasing version: ${env.RELEASE_VERSION}, Next dev version: ${env.NEXT_SNAPSHOT_VERSION}"

                    sh """
                        mvn release:clean release:prepare release:perform \
                          -DreleaseVersion=${env.RELEASE_VERSION} \
                          -DdevelopmentVersion=${env.NEXT_SNAPSHOT_VERSION} \
                          -B
                    """
                }
            }
        }

        stage('Update Lower Env Branches to SNAPSHOT') {
            when {
                expression { params.BRANCH_NAME == 'master' && params.RELEVER == 'major' }
            }
            steps {
                script {
                    def snapshotVersion = env.NEXT_SNAPSHOT_VERSION
                    def branchesToUpdate = ['develop', 'feature/lampfeature'] // 🔁 Add more as needed

                    branchesToUpdate.each { branch ->
                        echo "🔁 Updating ${branch} to version ${snapshotVersion}"

                        dir("tmp-${branch.replace('/', '_')}") {
                            git(
                                branch: "${branch}",
                                url: "${GIT_REPO}",
                                credentialsId: "${GIT_CREDENTIALS_ID}"
                            )

                            sh """
                                mvn versions:set -DnewVersion=${snapshotVersion}
                                mvn versions:commit
                                git config user.name "jenkins"
                                git config user.email "jenkins@ci.local"
                                git commit -am '🔄 Set version to ${snapshotVersion} after master release'
                                git push origin ${branch}
                            """
                        }
                    }
                }
            }
        }

        stage('Auto-Increment Version') {
            when {
                expression { !(params.BRANCH_NAME == 'master' && params.RELEVER == 'major') }
            }
            steps {
                script {
                    if (params.RELEVER == 'major') {
                        sh '''
                            mvn build-helper:parse-version versions:set \
                            -DnewVersion=\\${parsedVersion.nextMajorVersion}.0.0 \
                            versions:commit
                        '''
                    } else if (params.RELEVER == 'minor') {
                        sh '''
                            mvn build-helper:parse-version versions:set \
                            -DnewVersion=\\${parsedVersion.majorVersion}.\\${parsedVersion.nextMinorVersion}.0 \
                            versions:commit
                        '''
                    } else if (params.RELEVER == 'hotfix') {
                        sh '''
                            mvn build-helper:parse-version versions:set \
                            -DnewVersion=\\${parsedVersion.majorVersion}.\\${parsedVersion.minorVersion}.\\${parsedVersion.nextIncrementalVersion} \
                            versions:commit
                        '''
                    } else {
                        error "❌ Unsupported release version type: ${params.RELEVER}"
                    }
                }
            }
        }

        stage('Read Version from POM') {
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    FINAL_VERSION = pom.version
                    currentBuild.displayName = FINAL_VERSION
                    echo "📦 FINAL VERSION: ${FINAL_VERSION}"
                }
            }
        }

        stage('Store Version and Build Metadata') {
            steps {
                script {
                    def filePath = 'releases_build_map.log'
                    def build = env.BUILD_NUMBER
                    def version = FINAL_VERSION

                    def lines = []
                    if (fileExists(filePath)) {
                        lines = readFile(filePath).split('\n').collect { it.trim() }.findAll { it }
                    }

                    def entry = "${version}:${build}"
                    lines << entry
                    lines = lines.unique().takeRight(1)
                    writeFile file: filePath, text: lines.join('\n')

                    echo "✅ Stored release: ${FINAL_VERSION} and build number: ${build}"
                }
            }
        }

        stage('Build') {
            steps {
                echo "🛠️ Running Maven build"
                sh "mvn clean install -DskipTests=true"
            }
        }
    }

    post {
        always {
            echo "📦 Archiving artifacts: ${FINAL_VERSION}"
            archiveArtifacts artifacts: "**/target/*.jar, **/target/*.tar.gz", allowEmptyArchive: false
        }
        success {
            echo "✅ Build succeeded with version: ${FINAL_VERSION}"
        }
        failure {
            echo "❌ Pipeline failed. Check the logs for errors."
        }
    }
}
