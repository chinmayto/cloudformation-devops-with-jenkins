pipeline {
    agent any

    parameters {
            booleanParam(name: 'DEVELOPMENT', defaultValue: false, description: 'Check to deploy to Development environment')
            booleanParam(name: 'STAGING', defaultValue: false, description: 'Check to deploy to Staging environment')
            booleanParam(name: 'PRODUCTION', defaultValue: false, description: 'Check to deploy to Production environment')
    }

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
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]) {
                    dir('infrastructure') {
                        sh "aws cloudformation deploy --stack-name cfn-s3bucket --template-file create-cfn-template-bucket.yaml --region 'us-east-1' --capabilities CAPABILITY_IAM"
                        sh "aws s3 cp ../ s3://ct-cfn-files-for-stack/ --recursive"
                    }
                }
            }
        }

        stage('Development Deployment') {
            steps {
                script {
                    if (params.DEVELOPMENT) {
                       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]){
                            dir('infrastructure/development/') {
                                sh 'echo "=================Development Deployment=================="'
                                sh "aws cloudformation deploy --stack-name DeployDevelopmentStack --template-file root.yaml --region 'us-east-1' --capabilities CAPABILITY_IAM"
                            }
                        }
                    }
                }
            }
        }

        stage('Staging Deployment') {
            steps {
                script {
                    if (params.STAGING) {
                       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]){
                            dir('infrastructure/staging/') {
                                sh 'echo "=================Staging Deployment=================="'
                                sh "aws cloudformation deploy --stack-name DeployStagingStack --template-file root.yaml --region 'us-east-1' --capabilities CAPABILITY_IAM"
                            }
                        }
                    }
                }
            }
        }

        stage('Production Deployment') {
            steps {
                script {
                    if (params.PRODUCTION) {
                       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]){
                            dir('infrastructure/production/') {
                                sh 'echo "=================Production Deployment=================="'
                                sh "aws cloudformation deploy --stack-name DeployProductionStack --template-file root.yaml --region 'us-east-1' --capabilities CAPABILITY_IAM"
                            }
                        }
                    }
                }
            }
        }
    }
}