pipeline {
    agent any

    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }

    parameters {
        string(name: 'TARGET_HOST', defaultValue: 'plex_servers', description: 'Ansible inventory group or host to target')
        booleanParam(name: 'DRY_RUN', defaultValue: true, description: 'Run in check mode (no changes)')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Lint') {
            steps {
                sh 'ansible-lint site.yml || true'
            }
        }

        stage('Syntax Check') {
            steps {
                sh 'ansible-playbook site.yml -i inventory/hosts.yml --syntax-check'
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def checkFlag = params.DRY_RUN ? '--check' : ''
                    sh """
                        ansible-playbook site.yml \
                            -i inventory/hosts.yml \
                            -l ${params.TARGET_HOST} \
                            ${checkFlag} \
                            -v
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Plex deployment completed successfully.'
        }
        failure {
            echo 'Deployment failed. Check logs above.'
        }
    }
}
