pipeline {
    agent any   // Use any available Jenkins agent (your laptop in this case)

    tools {
        maven 'maven-3.9.6'   // Uses the Maven tool configured in Jenkins global config
        jdk 'JDK11'           // Uses the JDK 11 installation configured in Jenkins
    }

    parameters {
        choice(name: 'Action', choices: ['Build', 'Deploy'], description: 'Build or Deploy')  
        // Dropdown to choose whether to Build or Deploy
    }

    stages {

        stage('Initialize') {
            steps {
                bat 'mvn --version'   // Verifies Maven setup on Windows
                bat 'java -version'   // Verifies Java setup on Windows
            }
        }

        stage('Determine Version') {
            steps {
                script {
                    // Read the entire pom.xml content
                    def pomContent = readFile 'pom.xml'

                    // Use regular expression to find <version> inside <project>
                    def matcher = pomContent =~ '<project[^>]*>.*?<version>([^<]+)</version>'
                    def rawVersion = matcher ? matcher[0][1] : '0.0.1'

                    // Build a custom version string with BUILD_NUMBER and timestamp
                    def timestamp = new Date().format("yyyyMMdd.HHmmss")
                    def fullVersion = "${rawVersion}-SNAPSHOT-dev-${BUILD_NUMBER}-${timestamp}"
                    env.VERSION = fullVersion

                    // Replace the existing version in pom.xml with the new version
                    def updatedPom = pomContent.replaceFirst('<version>[^<]+</version>', "<version>${env.VERSION}</version>")

                    // Write the updated content back to pom.xml
                    writeFile file: 'pom.xml', text: updatedPom

                    echo "Generated Dev Version: ${env.VERSION}"
                }
            }
        }

        stage('Build') {
            when {
                expression { params.Action == 'Build' }
            }
            steps {
                bat 'mvn clean install -DskipTests=true'  // Compile and package the application
            }
        }

        stage('Deploy (Simulated)') {
            when {
                expression { params.Action == 'Deploy' }
            }
            steps {
                echo "Simulating deploy... No actual push."
                bat 'dir target\\*.jar || echo No jar built.'   // Show the built JAR file (if any)
            }
        }
    }

    post {
        always {
            echo "Lower environment build complete. Final Version: ${env.VERSION}"  // Always print the final version
        }
    }
}
