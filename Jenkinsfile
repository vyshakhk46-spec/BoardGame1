
pipeline {

    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    stages {

        stage('Git Checkout') {
            steps {
                echo '===== Checking out BoardGame source code ====='

                git(
                    branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/vyshakhk46-spec/BoardGame1.git'
                )
            }
        }

        stage('Compile') {
            steps {
                echo '===== Compiling Java application ====='

                sh '''
                    mvn clean compile
                '''
            }
        }

        stage('Test') {
            steps {
                echo '===== Running unit tests ====='

                sh '''
                    mvn test
                '''
            }
        }

        stage('File System Scan') {
            steps {
                echo '===== Running Trivy filesystem scan ====='

                sh '''
                    trivy fs \
                      --scanners vuln,secret \
                      --offline-scan \
                      --exit-code 0 \
                      --severity HIGH,CRITICAL \
                      .
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo '===== Running SonarQube analysis ====='

                withSonarQubeEnv('sonar') {
                    sh '''
                        mvn verify \
                          org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                          -Dsonar.projectKey=boardgame \
                          -Dsonar.projectName=BoardGame
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo '===== Checking SonarQube Quality Gate ====='

                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build') {
            steps {
                echo '===== Building Maven application ====='

                sh '''
                    mvn package -DskipTests
                '''
            }
        }

        stage('Publish Artifacts to Nexus') {
            steps {
                echo '===== Publishing JAR to Nexus ====='

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-cred',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        trap 'rm -f settings-nexus.xml' EXIT

                        cat > settings-nexus.xml <<EOF
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">

    <servers>
        <server>
            <id>maven-releases</id>
            <username>${NEXUS_USERNAME}</username>
            <password>${NEXUS_PASSWORD}</password>
        </server>
    </servers>

</settings>
EOF

                        mvn deploy \
                          -DskipTests \
                          -s settings-nexus.xml
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '===== Building Docker image ====='

                sh '''
                    docker build \
                      -t vysakh46/database-service-project:0.0.2 \
                      .
                '''
            }
        }

        stage('Docker Image Scan') {
            steps {
                echo '===== Running Trivy Docker image scan ====='

                sh '''
                    trivy image \
                      --exit-code 0 \
                      --severity HIGH,CRITICAL \
                      vysakh46/database-service-project:0.0.2
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                echo '===== Pushing Docker image to Docker Hub ====='

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        docker push vysakh46/database-service-project:0.0.2

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo '===== Deploying application to Kubernetes ====='

                sh '''
                    echo "===== Kubernetes Nodes ====="
                    kubectl get nodes

                    echo "===== Updating Deployment Image ====="
                    kubectl set image deployment/database-service \
                        database-service=vysakh46/database-service-project:0.0.2

                    echo "===== Waiting for Rollout ====="
                    kubectl rollout status deployment/database-service \
                        --timeout=120s

                    echo "===== Deployment Status ====="
                    kubectl get deployment database-service

                    echo "===== Pod Status ====="
                    kubectl get pods -l app=database-service

                    echo "===== Service Status ====="
                    kubectl get service database-service
                '''
            }
        }
    }

    post {

        always {
            script {

                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.currentResult ?: 'UNKNOWN'

                def bannerColor =
                    pipelineStatus.toUpperCase() == 'SUCCESS'
                    ? 'green'
                    : 'red'

                def body = """
                    <html>
                    <body>

                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">

                        <h2>${jobName} - Build ${buildNumber}</h2>

                        <div style="background-color: ${bannerColor}; padding: 10px;">
                            <h3 style="color: white;">
                                Pipeline Status: ${pipelineStatus.toUpperCase()}
                            </h3>
                        </div>

                        <p>
                            Check the
                            <a href="${BUILD_URL}">
                                Jenkins Console Output
                            </a>
                        </p>

                    </div>

                    </body>
                    </html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'shubhammukherji654@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}

