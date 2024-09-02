# Use Jenkins LTS image as the base image
FROM jenkins/jenkins:lts

# Copy the scripts to the Jenkins home directory
COPY --chown=jenkins:jenkins ./scripts/ ${JENKINS_HOME}/init.groovy.d/

# Copy the plugins.txt file to the Jenkins reference directory
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt

# Copy the scripts to the Jenkins home directory
COPY --chown=jenkins:jenkins ./relative_path_to_scripts/scripts/ ${JENKINS_HOME}/init.groovy.d/

# Copy the plugins.txt file to the Jenkins reference directory
COPY ./relative_path_to_plugins/plugins.txt /usr/share/jenkins/ref/plugins.txt

# Install Jenkins plugins
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

# Expose port 9001
EXPOSE 9001
