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
                    pip install --upgrade pip
                    pip install -r requirements-dev.txt
                '''
            }
        }
        stage('Lint') {
            steps { sh '. venv/bin/activate && flake8 .' }
        }
        stage('Unit Tests') {
            steps { sh '. venv/bin/activate && pytest -v' }
        }
        stage('Docker Build') {
            steps { sh 'docker build --target base -t aceest-fitness:${BUILD_NUMBER} .' }
        }
        stage('Docker Test') {
            steps { sh 'docker build --target test -t aceest-fitness:test . && docker run --rm aceest-fitness:test' }
        }
    }
    post {
        success { echo 'BUILD SUCCESSFUL' }
        failure { echo 'BUILD FAILED' }
    }
}
