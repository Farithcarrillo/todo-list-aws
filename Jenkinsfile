pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION  = 'us-east-1'
        STACK_NAME  = 'todo-list-aws-staging'
    }

    stages {

        stage('Get Code') {
            steps {
                checkout scm
                sh '''
                    set -e
                    echo "Branch:  $(git rev-parse --abbrev-ref HEAD)"
                    echo "Commit:  $(git rev-parse HEAD)"
                    echo "Workspace: $WORKSPACE"
                '''
            }
        }

        stage('Static Test') {
            steps {
                sh '''
                    set -e
                    python3 -m flake8 --format=pylint src/ > flake8.out || true
                    python3 -m bandit -r src/ -f custom \
                        --msg-template "{abspath}:{line}: [{test_id}] {msg}" \
                        -o bandit.out || true
                    echo "--- flake8.out ---"; cat flake8.out || true
                    echo "--- bandit.out ---"; cat bandit.out || true
                '''
                recordIssues(
                    tools: [
                        flake8(pattern: 'flake8.out'),
                        pyLint(name: 'Bandit', id: 'bandit', pattern: 'bandit.out')
                    ]
                )
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -e
                    rm -f samconfig.toml
                    git clone -b staging --depth 1 https://github.com/Farithcarrillo/todo-list-aws-config.git _cfg
                    cp _cfg/samconfig.toml samconfig.toml
                    rm -rf _cfg
                    sam validate --region ${AWS_REGION}
                    sam build
                    sam deploy \
                        --config-env staging \
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
                    echo "BASE_URL staging = ${env.BASE_URL}"
                    if (!env.BASE_URL?.startsWith('https://')) {
                        error("No se pudo obtener BaseUrlApi del stack ${env.STACK_NAME}")
                    }
                }
            }
        }

        stage('Rest Test') {
            steps {
                withEnv(["BASE_URL=${env.BASE_URL}"]) {
                    sh 'python3 -m pytest -m api test/integration/todoApiTest.py -v'
                }
            }
        }

        stage('Promote') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'github-token', gitToolName: 'Default')]) {
                    sh '''
                        set -e
                        git config user.email "jenkins@ci.local"
                        git config user.name  "Jenkins CI"
                        git fetch origin master:refs/remotes/origin/master
                        git fetch origin
                        git checkout -B master origin/master
                        # Merge develop en master con preferencia a develop en caso de conflicto.
                        git merge --no-ff origin/develop -m "Promote: develop -> master" -X theirs || true
                        # Restaurar los Jenkinsfile de master (no deben venir de develop).
                        git checkout origin/master -- Jenkinsfile Jenkinsfile_agentes 2>/dev/null || true
                        git add -A
                        git commit -m "Promote develop->master (preserva Jenkinsfile CD)" || echo "Sin cambios para commit"
                        git push origin master
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'flake8.out, bandit.out', allowEmptyArchive: true
        }
    }
}
