pipeline {
    agent {
        kubernetes {
            inheritFrom 'maven'        // our "maven" template (Helm): JDK 17 + Maven
            defaultContainer 'maven'   // every sh step runs in the maven container
        }
    }

    parameters {
        booleanParam(name: 'RUN_INTEGRATION_TESTS', defaultValue: true)
    }

    stages {
        stage('Test') {
            parallel {
                stage('Unit tests') {
                    steps {
                        sh 'chmod +x ./mvnw && ./mvnw test -D testGroups=unit'
                    }
                }
                stage('Integration tests') {
                    when {
                        expression { return params.RUN_INTEGRATION_TESTS }
                    }
                    steps {
                        sh 'chmod +x ./mvnw && ./mvnw test -D testGroups=integration'
                    }
                }
            }
        }
    }
}
