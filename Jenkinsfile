/* pipeline {
    agent any
    
    environment {
        DOCKER_HUB_REPO = "shivammitra/flask-hello-world"
        CONTAINER_NAME = "flask-hello-world"
        DOCKERHUB_CREDENTIALS=credentials('dockerhub-credentials')
    }
    
    stages { */
        /* We do not need a stage for checkout here since it is done by default when using "Pipeline script from SCM" option. */
        
        /* stage('Build') {
            steps {
                echo 'Building..'
                sh 'docker image build -t $DOCKER_HUB_REPO:latest .'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
                sh 'docker stop $CONTAINER_NAME || true'
                sh 'docker rm $CONTAINER_NAME || true'
                sh 'docker run --name $CONTAINER_NAME $DOCKER_HUB_REPO /bin/bash -c "pytest test.py && flake8"'
            }
        }
        stage('Push') {
            steps {
                echo 'Pushing image..'
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker push $DOCKER_HUB_REPO:latest'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying....'
                sh 'minikube kubectl -- apply -f deployment.yaml'
                sh 'minikube kubectl -- apply -f service.yaml'
            }
        }
    }
} */

pipeline {
    agent any

    environment {
        VENV_DIR = 'venv'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Setup Python Environment') {
            steps {
                echo 'Creating virtual environment and installing dependencies...'
                sh '''
                    python3 -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install pytest pytest-cov flake8
                '''
            }
        }

        stage('Lint') {
            steps {
                echo 'Running linters...'
                sh '''
                    . ${VENV_DIR}/bin/activate
                    flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics || true
                    flake8 . --count --max-line-length=120 --statistics || true
                '''
            }
        }

        stage('Test & Coverage') {
            steps {
                echo 'Running tests with coverage...'
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pytest --junitxml=test-results.xml \
                           --cov=. \
                           --cov-report=xml:coverage.xml \
                           --cov-report=term || true
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test-results.xml'
                }
            }
        }

        stage('SonarQube Scan') {
            steps {
                echo 'Running SonarQube analysis...'
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('MySonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Waiting for SonarQube Quality Gate...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs above.'
        }
    }
}
