pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        BUCKET_NAME = 'aishu-backup'
        CLUSTER_NAME = 'demo-cluster'
    }

    stages {

        stage('Install Velero CLI') {
            steps {
                sh '''
                if ! command -v velero &> /dev/null
                then
                  curl -L https://github.com/vmware-tanzu/velero/releases/latest/download/velero-linux-amd64.tar.gz | tar -xz
                  sudo mv velero*/velero /usr/local/bin/
                fi
                velero version
                '''
            }
        }

        stage('Terraform Init and Apply for Velero') {
            steps {
                dir('terraform/velero') {
                    sh 'terraform init'
                    sh 'terraform apply -auto-approve'
                }
            }
        }

        stage('Verify Velero Deployment') {
            steps {
                sh '''
                kubectl get pods -n velero
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
                    velero restore create restore-${env.BUILD_ID} --from-backup ${latestBackup}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Velero Backup and Restore completed successfully!"
        }
        failure {
            echo "Pipeline failed. Please check the steps."
        }
    }
}
