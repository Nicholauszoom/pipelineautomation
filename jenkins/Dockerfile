FROM jenkins/jenkins:lts-jdk21

# Install Docker CLI in Jenkins container
USER root
RUN apt-get update && apt-get install -y docker.io
USER jenkins
