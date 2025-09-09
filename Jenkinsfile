pipeline {
    
agent {
        kubernetes {
            label 'kube_m' // ✅ Must match the label in the pod template
            defaultContainer 'jnlp'
        }
    }


    tools {
        nodejs 'nodejs'
        maven 'maven'
        gradle 'gradle'
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
        skipDefaultCheckout()
        timestamps()
        disableResume()
        retry(0)
    }

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'gradel/fix_8.5', description: 'Branch to build')
        string(name: 'DOCKERHUBREPO', defaultValue: 'daggu1997/trainschedulegradle', description: 'Docker Hub repository to push the image')
        string(name: 'VERSION', defaultValue: 'latest', description: 'Version of the Docker image')
        string(name: 'DOCKER_HUB_CREDENTIALS_ID', defaultValue: 'docker', description: 'Credentials ID for Docker Hub')
        string(name: 'GITHUB_CREDENTIALS_ID', defaultValue: 'github', description: 'Credentials ID for GitHub access')
        string(name: 'GITHUB_REPO', defaultValue: 'Rajendra0609/cicd-pipeline-train-schedule-gradle', description: 'GitHub repository in owner/repo format')
        string(name: 'EMAIL_RECIPIENTS', defaultValue: 'rajendra.daggubati09@gmail.com,srirajendraprasaddaggubati@gmail.com', description: 'Comma-separated list of email recipients')
    }

    environment {
        DOCKER_HUB_CREDENTIALS_ID = 'docker'
        GIT_BRANCH = "${params.GIT_BRANCH}"
        GITHUB_CREDENTIALS_ID = 'github'
        GITHUB_REPO = 'Rajendra0609/cicd-pipeline-train-schedule-gradle'
        GITHUB_API_URL = 'https://api.github.com'
        EMAIL_RECIPIENTS = 'rajendra.daggubati09@gmail.com,srirajendraprasaddaggubati@gmail.com'
        TERM = 'xterm-256color'
        GITHUB_TOKEN = 'git_token'
        DOCKER_USER = 'docker_user'
        DOCKER_PASS = 'docker_token'
        SCANNER_HOME = tool 'sonar'
    }

    stages {
        stage('Checkout_startup') {
            steps {
                echo '🔄 cloing the code'
                checkout scm: [
                    $class: 'GitSCM',
                    branches: [[name: "${params.GIT_BRANCH}"]],
                    userRemoteConfigs: [[
                        url: "https://github.com/Rajendra0609/cicd-pipeline-train-schedule-gradle.git",
                        credentialsId: 'github',
                        name: 'origin'
                    ]]
                ]
            }
        }

        stage('Gradle_build') {
            steps {
                echo "🏗️  Running Build Process ...... 🔄⏳"
                sh '''
                ./gradlew build --no-daemon
                '''
                }
                post {
                    success {
                        echo "✅ Build succeeded! Artifacts archived. 📦"
                    }
                    failure {
                        echo "❌ Build failed! Please check the logs. 🪵"
                    }
                    always {
                        archiveArtifacts artifacts: 'dist/trainSchedule.zip'
                    }
            }
        }
        stage('Lynis_Scan') {
            steps {
                echo '🔍 Starting Lynis security scan...'
                sh '''
                    mkdir -p artifacts/lynis

                    # Run Lynis, allow it to fail softly (ignore systemd errors)
                    lynis audit system --quiet --report-file artifacts/lynis/lynis-report.log --no-colors || true

                    # Generate HTML report from stdout, again ignoring errors
                    lynis audit system | ansi2html > artifacts/lynis/lynis-report.html || true

                '''
                archiveArtifacts artifacts: 'artifacts/lynis/lynis-report.log', allowEmptyArchive: true
                archiveArtifacts artifacts: 'artifacts/lynis/lynis-report.html', allowEmptyArchive: true
                echo '✅ Lynis report published successfully.'
            }
        }

        stage('SonarQube_Scan') {
            steps {
                script {
                     withSonarQubeEnv('sonar') {
                        sh '''
                       
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=Sky-weather-application \
                        -Dsonar.sources=.
                    '''
                     }
                        echo '✅ SonarQube scan completed.'
                    }
                }
            }

        stage('Docker_Build') {
            steps {
                echo '🐳 Building Docker image...'
                script {
                    dockerImage = docker.build("${params.DOCKERHUBREPO}:${params.VERSION}", "-f Dockerfile .")
                }
                echo '✅ Docker image built successfully.'
            }
        }

        stage('Trivy') {
            steps {
                echo '🔍 Starting Trivy scan...'
                script {
                    def imageName = "${params.DOCKERHUBREPO}:${params.VERSION}"
                    sh """
                        trivy image --format json --output trivy-report.json --severity HIGH,CRITICAL ${imageName} || true
                    """
                }
                echo '✅ Trivy scan completed.'
            }

            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('Docker_Push') {
            steps {
                echo '🚀 Pushing Docker image to Docker Hub...'
                script {
                    docker.withRegistry('https://registry.hub.docker.com', "${DOCKER_HUB_CREDENTIALS_ID}") {
                        dockerImage.push("latest")
                    }
                }
                echo '✅ Docker image pushed successfully.'
            }
        }
    }

    post {
        success {
            echo 'Build & Deploy completed successfully!'
            mail to: "${EMAIL_RECIPIENTS}",
                 subject: "SUCCESS: ${env.JOB_NAME} [#${env.BUILD_NUMBER}]",
                 body: """\
The Jenkins Pipeline completed successfully.

🔗 Pipeline URL: ${env.BUILD_URL}
👷 Triggered by: ${currentBuild.getBuildCauses()[0].userName}

View the full job here: ${env.BUILD_URL}
"""
        }

        failure {
            script {
                def log = currentBuild.rawBuild.getLog(1000)
                def lastLines = log.takeRight(50).join('\n')

                def culprit = "Unknown"
                def changeAuthor = "Unknown"

                try {
                    changeAuthor = currentBuild.changeSets.collect { cs ->
                        cs.items.collect { it.author.fullName }
                    }.flatten().unique().join(', ')
                    culprit = currentBuild.getBuildCauses()[0].userName
                } catch (e) {
                    echo "Failed to determine author or trigger: ${e.message}"
                }

                mail to: "${EMAIL_RECIPIENTS}",
                     subject: "FAILURE: ${env.JOB_NAME} [#${env.BUILD_NUMBER}]",
                     body: """\
The Jenkins Pipeline has FAILED ❌

🔍 Failure Stage: See the Stage View or Blue Ocean for exact stage
👤 Git Committer(s): ${changeAuthor}
🚀 Triggered by: ${culprit}
🔗 Pipeline URL: ${env.BUILD_URL}

📄 Last 50 lines of console output:
--------------------------------------------------
${lastLines}
--------------------------------------------------

Please investigate the issue.
"""
            }
        }

        unstable {
            script {
                def log = currentBuild.rawBuild.getLog(1000)
                def lastLines = log.takeRight(50).join('\n')

                def culprit = "Unknown"
                def changeAuthor = "Unknown"

                try {
                    changeAuthor = currentBuild.changeSets.collect { cs ->
                        cs.items.collect { it.author.fullName }
                    }.flatten().unique().join(', ')
                    culprit = currentBuild.getBuildCauses()[0].userName
                } catch (e) {
                    echo "Failed to determine author or trigger: ${e.message}"
                }

                mail to: "${EMAIL_RECIPIENTS}",
                     subject: "UNSTABLE: ${env.JOB_NAME} [#${env.BUILD_NUMBER}]",
                     body: """\
The Jenkins Pipeline is UNSTABLE ⚠️

🔍 Potential Failure Stage: See the Stage View or Blue Ocean for exact stage
👤 Git Committer(s): ${changeAuthor}
🚀 Triggered by: ${culprit}
🔗 Pipeline URL: ${env.BUILD_URL}

📄 Last 50 lines of console output:
--------------------------------------------------
${lastLines}
--------------------------------------------------

Please investigate the warning.
"""
            }
        }

        always {
            cleanWs()
            echo 'Workspace cleaned'
        }
    }
}
