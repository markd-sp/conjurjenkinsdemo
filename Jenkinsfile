pipeline {
    agent any

    environment {
        GITHUB_REPO = 'https://github.com/markd-sp/conjurjenkinsdemo.git'
    }

    stages {

        stage('Retrieve Secrets from Conjur') {
            steps {
                script {
                    withCredentials([
                        conjurSecretCredential(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        conjurSecretCredential(credentialsId: 'AWS_SECRET_KEY',    variable: 'AWS_SECRET_KEY'),
                        conjurSecretCredential(credentialsId: 'AWS_REGION',        variable: 'AWS_REGION'),
                        conjurSecretCredential(credentialsId: 'S3_BUCKET',         variable: 'S3_BUCKET')
                    ]) {
                        echo "Secrets retrieved successfully"

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
                                BUCKET_NAME="${S3_BUCKET}"
                                REGION="${AWS_REGION}"
                                echo "Website URL: http://${BUCKET_NAME}.s3-website-${REGION}.amazonaws.com"
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
