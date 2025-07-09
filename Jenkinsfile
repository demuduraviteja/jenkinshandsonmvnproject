def FINAL_VERSION = '0'
def ENVIRONMENT = 'PROD'

pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
        // jdk 'java-11-openjdk'
    }

    environment {
        TIMESTAMP            = "${new Date().format('yyyyMMdd.HHmmss')}"
        GIT_REPO             = 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git'
        GIT_CREDENTIALS_ID   = 'github-pat'
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'master_version', description: 'Git branch to build')
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

        stage('Handle Versioning (Hybrid)') {
            steps {
                script {
                    echo "🔍 Parsing current version and computing next ${params.RELEVER} version..."

                    if (params.BRANCH_NAME.startsWith('master') || params.BRANCH_NAME.startsWith('hotfix')) {
                        // Manual version parsing for release plugin
                        def currentVersion = sh(
                            script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout | sed 's/-SNAPSHOT//'",
                            returnStdout: true
                        ).trim()

                        def parts = currentVersion.tokenize('.')
                        def major = parts[0].toInteger()
                        def minor = parts[1].toInteger()
                        def patch = parts[2].toInteger()

                        if (params.RELEVER == 'major') {
                            major += 1; minor = 0; patch = 0
                        } else if (params.RELEVER == 'minor') {
                            minor += 1; patch = 0
                        } else if (params.RELEVER == 'hotfix') {
                            patch += 1
                        } else {
                            error "❌ Unsupported RELEVER type: ${params.RELEVER}"
                        }

                        def releaseVersion = "${major}.${minor}.${patch}"
                        def snapshotVersion = "${releaseVersion}-SNAPSHOT"

                        env.RELEASE_VERSION = releaseVersion
                        env.NEXT_SNAPSHOT_VERSION = snapshotVersion

                        echo "🏷️ Releasing version: ${releaseVersion}, Next dev version: ${snapshotVersion}"

                        withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                            sh """
                                git config user.name "jenkins"
                                git config user.email "jenkins@ci.local"
                                git config credential.helper store
                                echo "https://${GIT_USER}:${GIT_TOKEN}@github.com" > ~/.git-credentials

                                mvn release:clean release:prepare release:perform \
                                  -DreleaseVersion=${releaseVersion} \
                                  -DdevelopmentVersion=${snapshotVersion} \
                                  -B

                                rm -f ~/.git-credentials
                            """
                        }

                    } else {
                        // Auto-increment using build-helper for lower env branches
                        echo "🔄 Using build-helper plugin for auto versioning"

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
                            error "❌ Unsupported RELEVER type: ${params.RELEVER}"
                        }
                    }
                }
            }
        }

        stage('Update Lower Env Branches to SNAPSHOT') {
            when {
                expression { params.BRANCH_NAME.startsWith('master') }
            }
            steps {
                script {
                    def snapshotVersion = env.NEXT_SNAPSHOT_VERSION
                    def branchesToUpdate = ['develop', 'feature/lampfeature']

                    branchesToUpdate.each { branch ->
                        echo "🔁 Updating ${branch} to version ${snapshotVersion}"

                        dir("tmp-${branch.replace('/', '_')}") {
                            git(
                                branch: "${branch}",
                                url: "${GIT_REPO}",
                                credentialsId: "${GIT_CREDENTIALS_ID}"
                            )

                            withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                                sh """
                                    mvn versions:set -DnewVersion=${snapshotVersion}
                                    mvn versions:commit
                                    git config user.name "jenkins"
                                    git config user.email "jenkins@ci.local"
                                    git commit -am '🔄 Set version to ${snapshotVersion} after master release'
                                    git config credential.helper store
                                    echo "https://${GIT_USER}:${GIT_TOKEN}@github.com" > ~/.git-credentials
                                    git push origin ${branch}
                                    rm -f ~/.git-credentials
                                """
                            }
                        }
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
