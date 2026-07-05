pipeline {
    agent any

    environment {
        VENV_DIR    = 'venv'
        STAGING_DIR = "${WORKSPACE}/staging_deploy"
        APP_PORT    = '8000'
        // Configure these two as Jenkins Credentials (Secret text) and reference them below,
        // rather than hardcoding real values here.
        MONGO_URI   = credentials('MONGO_URI')
        SECRET_KEY  = credentials('SECRET_KEY')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }

    triggers {
        // Backup poll in case the GitHub webhook can't reach this Jenkins instance
        pollSCM('H/5 * * * *')
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out source code..."
                checkout scm
            }
        }

        stage('Build - Install Dependencies') {
            steps {
                echo "Creating virtual environment and installing dependencies..."
                sh '''
                    python3 -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install pytest pytest-cov certifi
                '''
            }
        }

        stage('Test') {
            steps {
                echo "Running unit tests with pytest..."
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pytest test_app.py --junitxml=result.xml --cov=. --cov-report=xml || true
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'result.xml'
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                allOf {
                    expression { currentBuild.currentResult == 'SUCCESS' }
                    branch 'main'
                }
            }
            steps {
                echo "Deploying application to staging environment..."
                sh '''
                    . ${VENV_DIR}/bin/activate
                    mkdir -p ${STAGING_DIR}
                    rsync -a --exclude 'venv' --exclude '.git' ./ ${STAGING_DIR}/

                    # Write .env for the staging instance from Jenkins credentials
                    echo "MONGO_URI=${MONGO_URI}" > ${STAGING_DIR}/.env
                    echo "SECRET_KEY=${SECRET_KEY}" >> ${STAGING_DIR}/.env

                    # Stop any previously running staging instance on this port
                    fuser -k ${APP_PORT}/tcp || true
                    sleep 2

                    cd ${STAGING_DIR}
                    nohup ${WORKSPACE}/${VENV_DIR}/bin/python app.py > staging.log 2>&1 &
                    sleep 3
                    echo "Staging deployment complete. App should be reachable on port ${APP_PORT}."
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
            mail to: "${env.NOTIFY_EMAIL ?: 'your-email@example.com'}",
                 subject: "SUCCESS: Jenkins Build #${env.BUILD_NUMBER} - ${env.JOB_NAME}",
                 body: "Good news!\n\nBuild #${env.BUILD_NUMBER} for ${env.JOB_NAME} completed successfully.\n\nView details: ${env.BUILD_URL}"
        }
        failure {
            echo 'Pipeline failed.'
            mail to: "${env.NOTIFY_EMAIL ?: 'your-email@example.com'}",
                 subject: "FAILED: Jenkins Build #${env.BUILD_NUMBER} - ${env.JOB_NAME}",
                 body: "Build #${env.BUILD_NUMBER} for ${env.JOB_NAME} failed.\n\nCheck the console output: ${env.BUILD_URL}console"
        }
        always {
            cleanWs(patterns: [[pattern: 'venv/**', type: 'INCLUDE']])
        }
    }
}
