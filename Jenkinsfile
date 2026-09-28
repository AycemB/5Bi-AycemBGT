// Jenkinsfile - ROOT of the repo (next to /backend, /frontend, docker-compose.yml)
// Flow: Checkout -> Build -> Test (throwaway MySQL) -> SonarQube -> Docker build -> Deploy -> Verify

pipeline {
    agent any

    // Names must match Manage Jenkins > Tools
    tools {
        jdk   'JDK17'
        maven 'Maven3'
    }

    parameters {
        booleanParam(name: 'RUN_QUALITY_GATE', defaultValue: false,
                     description: 'Wait for the SonarQube Quality Gate (needs a SonarQube webhook to Jenkins)')
    }

    triggers {
        githubPush()
        // pollSCM('H/5 * * * *')   // use this instead if GitHub cannot reach Jenkins
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 40, unit: 'MINUTES')
    }

    environment {
        COMPOSE_PROJECT_NAME = 'gestion-projets'
        SONAR_PROJECT_KEY    = 'DevOps-AppGestionDesProjets'
        TEST_DB_CONTAINER    = 'mysql-ci-test'
        TEST_DB_PORT         = '3307'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm      // code comes from GitHub, NOT from a hard-coded /home/... folder
            }
        }

        stage('Check Environment') {
            steps {
                sh '''
                    echo "=== Jenkins user ===";  whoami
                    echo "=== Java ===";          java -version
                    echo "=== Maven ===";         mvn -version
                    echo "=== Docker ===";        docker --version
                    echo "=== Docker Compose ==="; docker compose version
                '''
            }
        }

        stage('Backend - Compile') {
            steps {
                dir('backend') {
                    sh 'mvn -B clean compile'
                }
            }
        }

        stage('Backend - Test') {
            steps {
                // Throwaway MySQL because @SpringBootTest needs a database
                sh '''
                    docker rm -f $TEST_DB_CONTAINER >/dev/null 2>&1 || true
                    docker run -d --name $TEST_DB_CONTAINER \
                        -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=test_db \
                        -p 127.0.0.1:$TEST_DB_PORT:3306 mysql:8.0

                    echo "Waiting for MySQL..."
                    for i in $(seq 1 30); do
                        docker exec $TEST_DB_CONTAINER mysqladmin ping -uroot -proot --silent && break
                        sleep 3
                    done
                '''
                dir('backend') {
                    sh '''
                        SPRING_DATASOURCE_URL="jdbc:mysql://127.0.0.1:$TEST_DB_PORT/test_db?allowPublicKeyRetrieval=true&useSSL=false" \
                        SPRING_DATASOURCE_USERNAME=root \
                        SPRING_DATASOURCE_PASSWORD=root \
                        mvn -B test
                    '''
                }
            }
            post {
                always {
                    junit testResults: 'backend/target/surefire-reports/*.xml', allowEmptyResults: true
                    sh 'docker rm -f $TEST_DB_CONTAINER || true'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        // SONAR_AUTH_TOKEN and SONAR_HOST_URL are injected by withSonarQubeEnv
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.projectName=${SONAR_PROJECT_KEY} \
                              -Dsonar.sources=backend/src/main/java,frontend/src \
                              -Dsonar.tests=backend/src/test/java \
                              -Dsonar.java.binaries=backend/target/classes \
                              -Dsonar.exclusions=**/node_modules/**,**/target/**,**/dist/**,**/*.spec.ts \
                              -Dsonar.token=\$SONAR_AUTH_TOKEN
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            when { expression { params.RUN_QUALITY_GATE } }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Deploy') {
            steps {
                // up -d recreates only what changed; DB volume is kept
                sh 'docker compose up -d --remove-orphans'
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    docker compose ps
                    for i in $(seq 1 30); do
                        if curl -sf http://localhost:8080/entreprise/all >/dev/null; then
                            echo "Backend OK"
                            curl -sf http://localhost:4300  >/dev/null && echo "Frontend OK"
                            exit 0
                        fi
                        echo "Waiting for backend ($i/30)..."
                        sleep 5
                    done
                    echo "Backend did not become healthy"
                    exit 1
                '''
            }
        }
    }

    post {
        success { echo 'Analyse SonarQube et deploiement reussis.' }
        failure {
            echo 'Le pipeline a echoue.'
            sh 'docker compose logs --tail=80 || true'
        }
    }
}
