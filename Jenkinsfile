// CD pipeline - rama master
// Despliega a produccion y ejecuta unicamente los tests de solo lectura
// para no alterar datos reales (no Create/Update/Delete en produccion).
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'us-east-1'
        STACK_NAME = 'todo-list-aws-production'
    }

    stages {

        stage('Get Code') {
            steps {
                checkout scm
                sh '''
                    set -e
                    whoami
                    hostname
                    echo "Branch:    $(git rev-parse --abbrev-ref HEAD)"
                    echo "Commit:    $(git rev-parse HEAD)"
                    echo "Workspace: $WORKSPACE"
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -e
                    sam validate --region ${AWS_REGION}
                    sam build
                    sam deploy \
                        --config-env production \
                        --resolve-s3 \
                        --no-confirm-changeset \
                        --no-fail-on-empty-changeset \
                        --region ${AWS_REGION}
                '''
                script {
                    env.BASE_URL = sh(
                        returnStdout: true,
                        script: "aws cloudformation describe-stacks --stack-name ${env.STACK_NAME} --query \"Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue\" --output text --region ${env.AWS_REGION}"
                    ).trim()
                    echo "BASE_URL production = ${env.BASE_URL}"
                    if (!env.BASE_URL?.startsWith('https://')) {
                        error("No se pudo obtener BaseUrlApi del stack ${env.STACK_NAME}")
                    }
                }
            }
        }

        stage('Rest Test') {
            steps {
                // Solo readonly: GET /todos. Si falla, aborta sin catchError.
                withEnv(["BASE_URL=${env.BASE_URL}"]) {
                    sh 'python3 -m pytest -m readonly test/integration/todoApiTest.py -v'
                }
            }
        }
    }
}
