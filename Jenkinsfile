pipeline {
    agent { label "${params.ENVIRONMENT}" }

    triggers {
        githubPush()
    }

    parameters {
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Version of the application')
        booleanParam(name: 'RUN_SONAR', defaultValue: false, description: 'Run SonarQube scan')
        choice(name: 'ENVIRONMENT', choices: ['Dev', 'staging', 'prod'], description: 'Deployment environment')
    }

    environment {
        SONARQUBE_ENV = 'sonarscanner'   // SonarQube server name in Manage Jenkins > System
        PROJECT_KEY   = 'Hello_flask'
        JFROG_URL     = 'http://35.175.171.123:8081/artifactory/python_project'  // change repo name
        APP_NAME      = 'hello-flask'
        APP_DIR       = '/opt/hello-flask'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'Git_cred',
                        url: 'https://github.com/<your-username>/hello-flask.git'
                    ]]
                )
            }
        }

        stage('Build') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest --junitxml=test-results.xml --cov=app --cov-report=xml
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test-results.xml'
                }
            }
        }

        stage('Sonar-scan') {
            when { expression { params.RUN_SONAR } }
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh 'sonar-scanner'
                }
            }
        }

        stage('Quality Gate') {
            when { expression { params.RUN_SONAR } }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            steps {
                sh """
                    mkdir -p dist
                    tar --exclude=venv --exclude=.git --exclude=dist -czf dist/${APP_NAME}-${params.VERSION}.tar.gz .
                """
                archiveArtifacts artifacts: 'dist/*.tar.gz', fingerprint: true
            }
        }

        stage('Push to JFrog') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Jrog_cred',
                                                  usernameVariable: 'JF_USER',
                                                  passwordVariable: 'JF_PASS')]) {
                    sh '''
                        curl -f -u "$JF_USER:$JF_PASS" \
                          -T "dist/${APP_NAME}-${VERSION}.tar.gz" \
                          "${JFROG_URL}/${APP_NAME}/${VERSION}/${APP_NAME}-${VERSION}.tar.gz"
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying to ${params.ENVIRONMENT}"
                sh """
                    set -e
                    sudo systemctl stop hello-flask || true
                    sudo mkdir -p ${APP_DIR}
                    sudo cp app.py requirements.txt ${APP_DIR}/
                    sudo python3 -m venv ${APP_DIR}/venv
                    sudo ${APP_DIR}/venv/bin/pip install -r ${APP_DIR}/requirements.txt
                    sudo systemctl start hello-flask
                    sleep 5
                    curl -f http://localhost:5000/health
                """
            }
        }
    }

    post {
        success { echo "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} deployed to ${params.ENVIRONMENT}" }
        failure { echo "FAILED: ${env.BUILD_URL}console" }
        cleanup { cleanWs() }
    }
}
