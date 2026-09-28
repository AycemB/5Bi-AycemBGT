// Jenkinsfile — place at the ROOT of the repository (same level as /backend and /frontend)
// Stack: Spring Boot 4.1 (Java 17, Maven) + Angular 22 (Node, npm)

// Works on Linux (sh) and Windows (bat) agents
def run(String cmd) {
    if (isUnix()) { sh cmd } else { bat cmd }
}

pipeline {
    agent any

    // Names must match Manage Jenkins > Tools
    tools {
        jdk    'JDK17'
        maven  'Maven3'
        nodejs 'NodeJS22'
    }

    triggers {
        githubPush()                       // GitHub webhook trigger
        // pollSCM('H/5 * * * *')          // fallback if webhook can't reach Jenkins (e.g. localhost)
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        BACKEND_DIR  = 'backend'
        FRONTEND_DIR = 'frontend'

        // The Spring context test (@SpringBootTest) needs MySQL.
        // These override application.properties (Spring relaxed binding).
        SPRING_DATASOURCE_URL      = 'jdbc:mysql://localhost:3306/test_db?createDatabaseIfNotExist=true'
        SPRING_DATASOURCE_USERNAME = 'root'
        SPRING_DATASOURCE_PASSWORD = 'root'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script { run('java -version && mvn -version && node -v && npm -v') }
            }
        }

        stage('Backend - Build') {
            steps {
                dir(env.BACKEND_DIR) {
                    script { run('mvn -B clean compile') }
                }
            }
        }

        stage('Backend - Test') {
            steps {
                dir(env.BACKEND_DIR) {
                    script { run('mvn -B test') }
                }
            }
            post {
                always {
                    junit testResults: "${env.BACKEND_DIR}/target/surefire-reports/*.xml",
                          allowEmptyResults: true
                }
            }
        }

        stage('Backend - Package') {
            steps {
                dir(env.BACKEND_DIR) {
                    script { run('mvn -B package -DskipTests') }
                }
            }
        }

        stage('Frontend - Install') {
            steps {
                dir(env.FRONTEND_DIR) {
                    script { run('npm ci') }
                }
            }
        }

        stage('Frontend - Test') {
            steps {
                dir(env.FRONTEND_DIR) {
                    script { run('npm test -- --watch=false') }
                }
            }
        }

        stage('Frontend - Build') {
            steps {
                dir(env.FRONTEND_DIR) {
                    script { run('npm run build') }
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: "${env.BACKEND_DIR}/target/*.jar, ${env.FRONTEND_DIR}/dist/**",
                                 fingerprint: true,
                                 allowEmptyArchive: false
            }
        }
    }

    post {
        success { echo 'Pipeline succeeded.' }
        failure { echo 'Pipeline failed - check the stage logs above.' }
        cleanup { cleanWs() }
    }
}
