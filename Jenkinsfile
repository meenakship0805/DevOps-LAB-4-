// Lab 4 / Q2 - CI/CD pipeline with automated test gates before deployment
// Student: Srishti Kainth (2301730175)
// Jenkins job: Pipeline script from SCM, repo github.com/srishhk/DevOps-LAB4, Script Path: Jenkinsfile
//
// Needs on the Jenkins machine: python3-venv, docker, kubectl + kubeconfig for minikube
// Jenkins credential (Manage Jenkins > Credentials, kind "Secret text"):
//   id = lab4-db-password   -> password for the throwaway test database
//
// Gates (any failure stops the pipeline; nothing is deployed):
//   1 lint  2 unit tests  3 integration tests on real PostgreSQL  4 coverage >= 80%
//   5 image build  6 deploy (blue-green)  7 post-deploy smoke test
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers { githubPush() }

    environment {
        IMAGE   = "lab4-app:${BUILD_NUMBER}"
        TEST_DB = "lab4-testdb-${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install -q --upgrade pip
                    pip install -q -r requirements-dev.txt
                '''
            }
        }

        stage('Gate 1: Lint') {
            steps { sh '. .venv/bin/activate && flake8 --max-line-length=120 app tests' }
        }

        stage('Gate 2: Unit tests') {
            steps {
                sh '. .venv/bin/activate && pytest tests/unit -v --junitxml=unit-results.xml'
            }
            post { always { junit 'unit-results.xml' } }
        }

        stage('Gate 3: Integration tests') {
            steps {
                // The password is injected by Jenkins and masked as **** in the log.
                withCredentials([string(credentialsId: 'lab4-db-password', variable: 'DB_PASSWORD')]) {
                    sh '''
                        docker run -d --name "$TEST_DB" -p 55432:5432 \
                          -e POSTGRES_DB=lab4 -e POSTGRES_USER=lab4 -e POSTGRES_PASSWORD="$DB_PASSWORD" \
                          postgres:16-alpine
                        for i in $(seq 1 30); do
                          docker exec "$TEST_DB" pg_isready -U lab4 -d lab4 && break
                          sleep 2
                        done
                        . .venv/bin/activate
                        DB_HOST=localhost DB_PORT=55432 pytest -m integration -v --junitxml=integration-results.xml
                    '''
                }
            }
            post {
                always {
                    junit 'integration-results.xml'
                    sh 'docker rm -f "$TEST_DB" || true'
                }
            }
        }

        stage('Gate 4: Coverage') {
            steps {
                sh '. .venv/bin/activate && pytest tests/unit -q --cov=app --cov-report=term-missing --cov-fail-under=80'
            }
        }

        stage('Build image') {
            steps {
                sh '''
                    docker build --build-arg APP_VERSION=b${BUILD_NUMBER} -t ${IMAGE} app
                    minikube image load ${IMAGE}
                '''
            }
        }

        stage('Deploy (blue-green)') {
            steps { sh 'bash scripts/blue_green_deploy.sh ${IMAGE}' }
        }

        stage('Smoke test') {
            steps {
                sh '''
                    URL=$(minikube service lab4-bg-live -n lab4 --url)
                    curl -fs "$URL/health"
                    curl -fs "$URL/" | grep "b${BUILD_NUMBER}"
                '''
            }
        }
    }

    post {
        success { echo "All gates passed - build ${BUILD_NUMBER} is live." }
        failure { echo "A gate failed - nothing was deployed (or the switch was rolled back)." }
        always  { cleanWs(deleteDirs: true, notFailBuild: true) }
    }
}
