pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                // Clean workspace before cloning (optional)
                deleteDir()

                // Clone the Git repository
                git branch: 'main',
                    url: 'https://github.com/chinmayto/cloudformation-devops-with-jenkins.git'

                sh "ls -lart"
            }
        }

        stage('Create CFN Template S3 Bucket') {
            steps {
                dir('infrastructure') {
                    sh "aws cloudformation deploy --stack-name cfn-s3bucket --template-file create-cfn-template-bucket.yaml --region 'us-east-1' -capabilities CAPABILITY_IAM"
                    sh "aws s3 cp . s3://ct-cfn-files-for-stack/ --recursive"
                }
            }
        }
    }
}