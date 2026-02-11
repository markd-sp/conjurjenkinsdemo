pipeline {
    agent any

    environment {
        GITHUB_REPO = 'git@github.com:markd-sp/conjurdemo.git'
    }

    stages {

        stage('Retrieve Secrets from Conjur') {
            steps {
                script {
                    withCredentials([
                        conjurSecretCredential(credentialsId: 'GITHUB_SSH_KEY',    variable: 'GITHUB_SSH_KEY'),
                        conjurSecretCredential(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        conjurSecretCredential(credentialsId: 'AWS_SECRET_KEY',    variable: 'AWS_SECRET_KEY'),
                        conjurSecretCredential(credentialsId: 'AWS_REGION',        variable: 'AWS_REGION'),
                        conjurSecretCredential(credentialsId: 'S3_BUCKET',         variable: 'S3_BUCKET')
                    ]) {
                        echo "Secrets retrieved successfully"

                        stage('Setup SSH Agent') {
                            sh '''
                                eval $(ssh-agent -s)

                                echo "SSH_AUTH_SOCK=${SSH_AUTH_SOCK}" > "${WORKSPACE}/.ssh-agent-env"
                                echo "SSH_AGENT_PID=${SSH_AGENT_PID}" >> "${WORKSPACE}/.ssh-agent-env"

                                TMPKEY=$(mktemp)
                                
                                set +x
                                printf '%s\n' "${GITHUB_SSH_KEY}" > "${TMPKEY}"
                                set -x
                                
                                chmod 600 "${TMPKEY}"

                                ssh-add "${TMPKEY}"
                                rm -f "${TMPKEY}"

                                # echo "Loaded keys:"
                                # ssh-add -l

                                ssh-keyscan -H github.com > "${WORKSPACE}/.github-known-hosts" 2>/dev/null
                                # echo "Known hosts line count: $(wc -l < "${WORKSPACE}/.github-known-hosts")"

                                echo "SSH agent ready ✓"
                            '''
                        }

                        stage('Verify SSH Connection to GitHub') {
                            sh '''
                                export $(cat "${WORKSPACE}/.ssh-agent-env" | xargs)

                                KNOWN_HOSTS="${WORKSPACE}/.github-known-hosts"

                                SSH_OUTPUT=$(ssh -T git@github.com \
                                    -o UserKnownHostsFile="${KNOWN_HOSTS}" \
                                    -o StrictHostKeyChecking=yes \
                                    2>&1 || true)

                                echo "GitHub response: ${SSH_OUTPUT}"

                                if echo "${SSH_OUTPUT}" | grep -q "successfully authenticated"; then
                                    echo "SSH verification passed ✓"
                                else
                                    echo "SSH verification failed — check public key is added to GitHub"
                                    exit 1
                                fi
                            '''
                        }

                        stage('Verify AWS Connection') {
                            sh '''
                                set +x
                                export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
                                export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_KEY}"
                                export AWS_DEFAULT_REGION="${AWS_REGION}"

                                aws sts get-caller-identity
                                if [ $? -ne 0 ]; then
                                    echo "AWS credential verification failed"
                                    exit 1
                                fi
                                echo "AWS credentials verified ✓"

                                aws s3 ls s3://${S3_BUCKET}
                                if [ $? -ne 0 ]; then
                                    echo "S3 bucket verification failed"
                                    exit 1
                                fi
                                echo "S3 bucket accessible ✓"
                            '''
                        }

                        stage('Checkout Repo') {
                            sh '''
                                export $(cat "${WORKSPACE}/.ssh-agent-env" | xargs)

                                KNOWN_HOSTS="${WORKSPACE}/.github-known-hosts"

                                GIT_SSH_COMMAND="ssh -o UserKnownHostsFile=${KNOWN_HOSTS} -o StrictHostKeyChecking=yes" \
                                git clone ${GITHUB_REPO} "${WORKSPACE}/repo"
                            '''
                        }

                        stage('Deploy to S3') {
                            sh '''
                                set +x
                                export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
                                export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_KEY}"
                                export AWS_DEFAULT_REGION="${AWS_REGION}"

                                aws s3 sync "${WORKSPACE}/repo/" s3://${S3_BUCKET}/ \
                                    --exclude ".git/*" \
                                    --exclude "Jenkinsfile" \
                                    --exclude "README.md"

                                echo "Deployment complete ✓"
                            '''
                        }

                    }
                }
            }
        }

    }

    post {
        always {
            sh '''
                if [ -f "${WORKSPACE}/.ssh-agent-env" ]; then
                    export $(cat "${WORKSPACE}/.ssh-agent-env" | xargs)
                    ssh-agent -k || true
                fi
                rm -f "${WORKSPACE}/.ssh-agent-env" "${WORKSPACE}/.github-known-hosts"
                rm -rf "${WORKSPACE}/repo"
            '''
            cleanWs()
            echo "Cleanup complete ✓"
        }
        success {
            echo '✓ Deployment succeeded!'
        }
        failure {
            echo '✗ Deployment failed — check console output for details'
        }
    }
}
