pipeline {
    agent any
    
    stages {
        stage('Get Code') {
            steps {
                // Etapa “Get Code” para descargar el código fuente del repositorio 
                git branch: 'develop', url: 'https://github.com/apastorsales/todo-list-aws.git'
            }
        }
        
        stage('Static tests') {
            steps {
                // Etapa “Static Test” para la ejecución de las pruebas de análisis estático.
                sh '''
                    flake8 --exit-zero --format=pylint src > flake8.out
                    bandit -r src > bandit.out || true
                '''
                
                recordIssues tools: [
                    flake8(name: 'Flake8', pattern: 'flake8.out'),
                    pyLint(name: 'Bandit', pattern: 'bandit.out')    
                ]
            }
        }
        
        stage('Deploy') {
            steps {
                // Etapa de despliegue SAM (“Deploy”)
                sh '''
                    sam build
                    sam validate --region us-east-1
                    sam deploy \
                        --config-env staging \
                        --no-confirm-changeset \
                        --no-fail-on-empty-changeset
                '''
            }
        }
        
        stage('Rest tests') {
            steps {
                // Etapa “Rest Test” para la ejecución de las pruebas de integración, sobre el entorno recién levantado.
                sh '''
                    export BASE_URL=$(aws cloudformation describe-stacks \
                      --stack-name todo-list-aws-staging \
                      --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' \
                      --region us-east-1 \
                      --output text)
                    
                    pytest --junitxml=result-rest.xml test/integration/todoApiTest.py
                '''
                
                junit 'result-rest.xml'
            }
        }
        
        stage('Promote') {
            // Etapa “Promote” para marcar la versión como “Release” y ser desplegada en producción.
            when {
                branch 'develop'
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        set -e
                        
                        git config user.email "jenkins@aps.com"
                        git config user.name "Jenkins[APS] CI"
                        
                        git fetch origin
                        
                        git checkout master
                        git pull origin master
                        
                        git merge develop --no-ff -m "Release version"
                        
                        git push https://${GIT_USER}:${GIT_TOKEN}@github.com/apastorsales/todo-list-aws.git master
                    '''
                }
            }
        }
    }
}