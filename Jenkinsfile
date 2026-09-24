pipeline {
    agent {
        kubernetes {
            // Pod: jnlp + Maven/Java 11 (Quarkus 1.12), everything in /tmp/agent (OpenShift fix)
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp
      image: jenkins/inbound-agent:latest-jdk17
      workingDir: /tmp/agent
      env:
        - { name: HOME, value: /tmp/agent }
      resources:
        requests: { cpu: 100m, memory: 256Mi }
        limits:   { cpu: 500m, memory: 512Mi }
    - name: maven
      image: maven:3.9-eclipse-temurin-11
      command: ["sleep"]
      args: ["infinity"]
      workingDir: /tmp/agent
      env:
        - { name: HOME, value: /tmp/agent }
        - { name: MAVEN_CONFIG, value: "" }
        - { name: MAVEN_OPTS, value: "-Duser.home=/tmp/agent" }
      resources:
        requests: { cpu: 250m, memory: 512Mi }
        limits:   { cpu: 1, memory: 1536Mi }
'''
            defaultContainer 'maven'
        }
    }

    parameters {
        booleanParam(name: 'RUN_INTEGRATION_TESTS', defaultValue: true)
    }

    stages {
        stage('Compile') {
            steps {
                // download dependencies + compile once, before the parallel stages
                sh 'mvn -B -DskipTests test-compile'
            }
        }

        stage('Test') {
            parallel {
                stage('Unit tests') {
                    steps {
                        sh 'mvn -B test -D testGroups=unit'
                    }
                }
                stage('Integration tests') {
                    when {
                        expression { return params.RUN_INTEGRATION_TESTS }
                    }
                    steps {
                        sh 'mvn -B test -D testGroups=integration'
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    try {
                        sh 'mvn -B package -D skipTests'
                    } catch (ex) {
                        echo "Error while generating JAR file"
                        throw ex
                    }
                }
            }
        }
    }
}
