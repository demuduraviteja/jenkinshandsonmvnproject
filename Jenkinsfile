pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
       // jdk 'JDK11'
    }

    parameters {
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/demuduraviteja/jenkinshandsonmvnproject.git', description: 'Git repository URL')
        string(name: 'BRANCH_NAME', defaultValue: '', description: 'Git branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'sit'], description: 'Target environment')
        // Uncomment below if you want manual Deploy trigger only
        // choice(name: 'Action', choices: ['Build', 'Deploy'], description: 'Choose Build or Deploy')
        string(name: 'Nexus_Cred_ID', defaultValue: '', description: 'Jenkins Credentials ID for Nexus')
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
                sh 'mvn --version'
                //sh 'java -version'
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
                    def suffix = (params.ENVIRONMENT == 'sit') 
                        ? "SNAPSHOT-${env.BUILD_NUMBER}-${env.TIMESTAMP}" 
                        : "SNAPSHOT-dev-${env.BUILD_NUMBER}-${env.TIMESTAMP}"

                    def mavenVersionCommand = ""

                    if (params.BRANCH_NAME.startsWith('feature') || params.BRANCH_NAME.startsWith('develop')) {
                        mavenVersionCommand = """
                            mvn build-helper:parse-version versions:set \\
                                -DnewVersion=\${parsedVersion.majorVersion}.\${parsedVersion.nextMinorVersion}.0-${suffix} \\
                                versions:commit
                        """
                        echo "📈 Feature/Develop branch: Bumping minor version (patch set to 0)"
                    } else if (params.BRANCH_NAME.startsWith('hotfix') || params.BRANCH_NAME.startsWith('bugfix')) {
                        mavenVersionCommand = """
                            mvn build-helper:parse-version versions:set \\
                                -DnewVersion=\${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextIncrementalVersion}-${suffix} \\
                                versions:commit
                        """
                        echo "🔧 Hotfix/Bugfix branch: Bumping patch version"
                    } else {
                        error "❌ Unsupported branch type for versioning: ${params.BRANCH_NAME}"
                    }

                    sh mavenVersionCommand

                    def pom = readMavenPom file: 'pom.xml'
                    def finalVersion = pom.version
                    echo "📦 Final Version set in pom.xml: ${finalVersion}"

                    writeFile file: 'build_version.txt', text: finalVersion
                    currentBuild.displayName = finalVersion
                }
            }
        }

        stage('Build') {
            steps {
                echo "🛠 Running Maven build"
                sh 'mvn clean install -DskipTests=true'
            }
        }

        stage('Deploy to Nexus') {
            when {
                expression { params.ENVIRONMENT == 'sit' } // Add: && params.Action == 'Deploy' if you use Action param
            }
            steps {
                echo "🚀 Deploying to Nexus for environment: ${params.ENVIRONMENT}"
                withCredentials([usernamePassword(
                    credentialsId: params.Nexus_Cred_ID,
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        echo "Creating Maven settings.xml"
                        cat > settings.xml <<EOF
<settings>
  <servers>
    <server>
      <id>nexus-snapshots</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
  </servers>
</settings>
EOF

                        echo "🔄 Running mvn deploy"
                        mvn -s settings.xml deploy -DskipTests=true
                    '''
                }
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
