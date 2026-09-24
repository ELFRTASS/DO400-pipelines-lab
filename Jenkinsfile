pipeline {
    agent {
        kubernetes {
            // One pod: jnlp + Maven/Java 11 (Quarkus 1.12), everything in /tmp/agent (OpenShift fix)
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

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    parameters {
        booleanParam(name: 'RUN_INTEGRATION_TESTS', defaultValue: true,  description: 'Run the integration tests')
        booleanParam(name: 'RUN_PERF_TESTS',        defaultValue: true,  description: 'Run the performance stage')
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'staging', 'prod', 'none'], description: 'Deployment target (prod asks for approval)')
    }

    environment {
        APP_NAME    = 'shopping-cart'
        APP_VERSION = "1.0.${env.BUILD_NUMBER}"
    }

    stages {

        // ───────────── 1. INIT (sequential) ─────────────
        stage('Init') {
            steps {
                echo "App: ${APP_NAME} v${APP_VERSION} | branch: ${env.BRANCH_NAME} | target: ${params.DEPLOY_ENV}"
                sh 'java -version && mvn -v && ls -la'
            }
        }

        // ───────────── 2. COMPILE (real) ─────────────
        stage('Compile') {
            steps {
                sh 'mvn -B -DskipTests test-compile'
            }
        }

        // ───────────── 3. QUALITY GATES (5 parallel) ─────────────
        stage('Quality Gates') {
            failFast true
            parallel {
                stage('Unit tests') {
                    steps { sh 'mvn -B test -D testGroups=unit' }
                }
                stage('Integration tests') {
                    when { expression { return params.RUN_INTEGRATION_TESTS } }
                    steps { sh 'mvn -B test -D testGroups=integration' }
                }
                stage('Code style') {
                    steps { sh 'echo "Checking code style..."; sleep 3; echo "Style OK"' }
                }
                stage('Static analysis') {
                    steps { sh 'echo "Running static analysis..."; sleep 4; echo "0 bugs, 0 vulnerabilities"' }
                }
                stage('License check') {
                    steps { sh 'echo "Checking dependency licenses..."; sleep 2; echo "All licenses allowed"' }
                }
            }
        }

        // ───────────── 4. SECURITY (parallel + sequential sub-stages) ─────────────
        stage('Security') {
            parallel {
                stage('Dependency scan') {
                    stages {
                        stage('Resolve deps') {
                            steps { sh 'echo "Resolving dependency tree..."; sleep 2' }
                        }
                        stage('Analyze deps') {
                            steps { sh 'echo "Checking CVE database..."; sleep 3; echo "No critical CVE"' }
                        }
                    }
                }
                stage('Secrets scan') {
                    steps { sh 'echo "Scanning for hard-coded secrets..."; sleep 3; echo "No secrets found"' }
                }
                stage('Container scan') {
                    stages {
                        stage('Pull base image') {
                            steps { sh 'echo "Pulling ubi8/openjdk-11..."; sleep 2' }
                        }
                        stage('Scan base image') {
                            steps { sh 'echo "Scanning image layers..."; sleep 3; echo "Image OK"' }
                        }
                    }
                }
            }
        }

        // ───────────── 5. COMPATIBILITY MATRIX (3 x 2 - 1 = 5 cells) ─────────────
        stage('Compatibility Matrix') {
            matrix {
                axes {
                    axis {
                        name 'JAVA_VERSION'
                        values '11', '17', '21'
                    }
                    axis {
                        name 'DATABASE'
                        values 'postgresql', 'mysql'
                    }
                }
                excludes {
                    exclude {
                        axis { name 'JAVA_VERSION'; values '21' }
                        axis { name 'DATABASE';     values 'mysql' }
                    }
                }
                stages {
                    stage('Setup env') {
                        steps { sh 'echo "Starting $DATABASE with Java $JAVA_VERSION..."; sleep 2' }
                    }
                    stage('Run compat tests') {
                        steps { sh 'echo "Compat tests: Java $JAVA_VERSION + $DATABASE"; sleep 3; echo "PASS"' }
                    }
                }
            }
        }

        // ───────────── 6. PACKAGE (real, try/catch) ─────────────
        stage('Package') {
            steps {
                script {
                    try {
                        sh 'mvn -B package -D skipTests'
                    } catch (ex) {
                        echo "Error while generating JAR file"
                        throw ex
                    }
                }
                archiveArtifacts artifacts: 'target/*.jar, target/quarkus-app/**', allowEmptyArchive: true
            }
        }

        // ───────────── 7. IMAGES (4 parallel) ─────────────
        stage('Images') {
            parallel {
                stage('Image: api') {
                    steps { sh 'echo "Building image api:${APP_VERSION}"; sleep 3' }
                }
                stage('Image: worker') {
                    steps { sh 'echo "Building image worker:${APP_VERSION}"; sleep 4' }
                }
                stage('Image: frontend') {
                    steps { sh 'echo "Building image frontend:${APP_VERSION}"; sleep 3' }
                }
                stage('Image: migrations') {
                    steps { sh 'echo "Building image migrations:${APP_VERSION}"; sleep 2' }
                }
            }
        }

        // ───────────── 8. E2E MATRIX (3 x 2 - 1 = 5 cells) ─────────────
        stage('E2E Matrix') {
            matrix {
                axes {
                    axis {
                        name 'BROWSER'
                        values 'chrome', 'firefox', 'edge'
                    }
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'windows'
                    }
                }
                excludes {
                    exclude {
                        axis { name 'BROWSER';  values 'edge' }
                        axis { name 'PLATFORM'; values 'linux' }
                    }
                }
                stages {
                    stage('Browser tests') {
                        steps { sh 'echo "E2E on $BROWSER / $PLATFORM"; sleep 3; echo "12/12 scenarios passed"' }
                    }
                }
            }
        }

        // ───────────── 9. PERFORMANCE (3 parallel, optional) ─────────────
        stage('Performance') {
            when { expression { return params.RUN_PERF_TESTS } }
            parallel {
                stage('Load test') {
                    steps { sh 'echo "500 users for 30s..."; sleep 4; echo "p95 = 180ms"' }
                }
                stage('Stress test') {
                    steps { sh 'echo "Ramping up to breaking point..."; sleep 5; echo "Max 2300 req/s"' }
                }
                stage('Soak test') {
                    steps { sh 'echo "Long-running stability check..."; sleep 4; echo "No memory leak"' }
                }
            }
        }

        // ───────────── 10. DEPLOY MATRIX (3 x 2 = 6 cells) ─────────────
        stage('Deploy Matrix') {
            when { expression { return params.DEPLOY_ENV != 'none' } }
            matrix {
                axes {
                    axis {
                        name 'REGION'
                        values 'eu-west', 'us-east', 'africa-north'
                    }
                    axis {
                        name 'COMPONENT'
                        values 'api', 'worker'
                    }
                }
                stages {
                    stage('Deploy') {
                        steps { sh 'echo "Deploying $COMPONENT to $REGION ($DEPLOY_ENV)"; sleep 2' }
                    }
                    stage('Verify') {
                        steps { sh 'echo "Health check $COMPONENT@$REGION"; sleep 1; echo "healthy"' }
                    }
                }
            }
        }

        // ───────────── 11. PROD APPROVAL (only for prod) ─────────────
        stage('Prod approval') {
            when { expression { return params.DEPLOY_ENV == 'prod' } }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: "Promote ${APP_NAME} v${APP_VERSION} to PROD?", ok: 'Promote'
                }
            }
        }

        // ───────────── 12. SMOKE TESTS (3 parallel) ─────────────
        stage('Smoke tests') {
            when { expression { return params.DEPLOY_ENV != 'none' } }
            parallel {
                stage('Smoke: health') {
                    steps { sh 'echo "GET /q/health -> 200"; sleep 1' }
                }
                stage('Smoke: api') {
                    steps { sh 'echo "GET /cart -> 200"; sleep 2' }
                }
                stage('Smoke: ui') {
                    steps { sh 'echo "Home page loads"; sleep 2' }
                }
            }
        }

        // ───────────── 13. RELEASE (sequential) ─────────────
        stage('Release') {
            steps {
                sh 'echo "Tagging v${APP_VERSION}"; echo "Generating release notes..."; sleep 2'
            }
        }
    }

    post {
        success { echo "✅ ${APP_NAME} v${APP_VERSION} delivered to ${params.DEPLOY_ENV}" }
        failure { echo "❌ Pipeline failed - check the red stage in Pipeline Overview" }
        always  { echo "Duration: ${currentBuild.durationString}" }
    }
}
