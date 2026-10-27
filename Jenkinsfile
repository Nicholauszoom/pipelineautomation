pipeline {
    agent any
    tools { maven 'Maven3'; jdk 'JDK17' }

    environment {
        REGISTRY   = 'registry.example.com'
        IMAGE_NAME = 'bayport/mandate-backend'
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Unit Test') {
            steps { sh 'mvn clean verify' }
            post { always { junit 'target/surefire-reports/*.xml' } }
        }

       stage('SonarQube Analysis') {
           steps {
               withSonarQubeEnv('sonar') {
                   sh '''
                     mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                       -Dsonar.projectKey=pipelineautomation \
                       -Dsonar.qualitygate.wait=true
                   '''
               }
           }
       }


        stage('Dependency Scan') {
            steps {
                sh 'mvn org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=8'
            }
        }

        stage('Build Image') {
            steps { sh "docker build -t $REGISTRY/$IMAGE_NAME:$IMAGE_TAG ." }
        }

        stage('Image Scan (Trivy)') {
            steps {
                sh """
                  docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image --exit-code 1 \
                    --severity HIGH,CRITICAL $REGISTRY/$IMAGE_NAME:$IMAGE_TAG
                """
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-creds',
                        usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh """
                      echo "$P" | docker login $REGISTRY -u "$U" --password-stdin
                      docker push $REGISTRY/$IMAGE_NAME:$IMAGE_TAG
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                sshagent(['deploy-ssh']) {
                    sh """
                      ssh -o StrictHostKeyChecking=no deploy@staging-server '
                        docker pull $REGISTRY/$IMAGE_NAME:$IMAGE_TAG &&
                        docker rm -f mandate-backend || true &&
                        docker run -d --name mandate-backend -p 8080:8080 \
                          -e SPRING_PROFILES_ACTIVE=staging \
                          $REGISTRY/$IMAGE_NAME:$IMAGE_TAG'
                    """
                }
            }
        }

        stage('Approve Production') {
            when { branch 'main' }
            steps { input message: 'Deploy to production?', submitter: 'admin,release-managers' }
        }

        stage('Deploy to Production') {
            when { branch 'main' }
            steps {
                sshagent(['deploy-ssh']) {
                    sh "ssh deploy@prod-server 'docker pull $REGISTRY/$IMAGE_NAME:$IMAGE_TAG && docker compose up -d'"
                }
            }
        }
    }

    post {
        always  { bat 'docker image prune -f' }
        failure { echo 'Pipeline failed. Check the logs.' }
    }
}