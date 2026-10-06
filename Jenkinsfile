// Linux Jenkins agent. Docker stages run only if Docker is available on the agent.
pipeline {
    agent any
    options { timestamps() }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Clean Build Environment') {
            steps {
                sh '''
                    rm -rf venv
                    python3 -m venv venv
                    . venv/bin/activate
                    python -m pip install --upgrade pip
                    python -m pip install -r requirements-dev.txt
                '''
            }
        }
        stage('Compile Check') {
            steps { sh '. venv/bin/activate && python -m py_compile app.py' }
        }
        stage('Lint') {
            steps { sh '. venv/bin/activate && python -m flake8 .' }
        }
        stage('Unit Tests') {
            steps { sh '. venv/bin/activate && python -m pytest -v' }
        }
        stage('Docker Build & Test') {
            when { expression { sh(script: 'command -v docker', returnStatus: true) == 0 } }
            steps {
                sh 'docker build --target base -t aceest-fitness:${BUILD_NUMBER} .'
                sh 'docker build --target test -t aceest-fitness:test .'
                sh 'docker run --rm aceest-fitness:test'
            }
        }
    }
    post {
        success { echo 'BUILD SUCCESSFUL' }
        failure { echo 'BUILD FAILED' }
    }
}