pipeline {
    agent any

    environment {
        AWS_REGION   = 'us-east-1'
        BUCKET_NAME  = 'aishu-backup'
        CLUSTER_NAME = 'demo-cluster'
        VELERO_VERSION = 'v1.13.1'
    }

    stages {

        stage('Install Velero CLI') {
            steps {
                sh '''
                if ! command -v velero &> /dev/null; then
                    echo "Downloading Velero CLI..."
                    curl -LO https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz
                    tar -xvzf velero-${VELERO_VERSION}-linux-amd64.tar.gz
                    sudo mv velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/
                fi

                echo "Velero CLI version:"
                velero version
                '''
            }
        }

        stage('Terraform Init and Apply for Velero') {
            steps {
                dir('terraform/velero') {
                    sh '''
                    terraform init
                    terraform apply -auto-approve
                    '''
                }
            }
        }

        stage('Verify Velero Deployment') {
            steps {
                sh '''
                echo "Checking Velero pods..."
                kubectl get pods -n velero

                echo "Checking Velero deployment..."
                kubectl get deployment velero -n velero
                '''
            }
        }

        stage('Create Velero Backup') {
            steps {
                script {
                    def backupName = "eks-backup-${env.BUILD_ID}"
                    sh """
                    velero backup create ${backupName} --include-namespaces default --wait
                    echo "Backup '${backupName}' created."
                    """
                }
            }
        }

        stage('Check Backup Status') {
            steps {
                sh 'velero backup get'
            }
        }

        stage('Manual Approval for Restore') {
            steps {
                input message: 'Do you want to perform a restore now?', ok: 'Yes, restore!'
            }
        }

        stage('Restore from Backup') {
            steps {
                script {
                    def latestBackup = sh(script: "velero backup get -o jsonpath='{.items[-1:].metadata.name}'", returnStdout: true).trim()
                    sh """
                    echo "Restoring from backup: ${latestBackup}"
                    velero restore create restore-${env.BUILD_ID} --from-backup ${latestBackup}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Velero Backup and Restore completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Please check the steps."
        }
    }
}
