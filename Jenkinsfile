pipeline {
    agent any
    environment {
        REGION = 'us-east-1'
        LAUNCH_TEMPLATE_NAME = 'LT-1'
        ASG_NAME = 'Asg-test'
        AMI_NAME_PREFIX = 'test'
    }
    stages {
        stage('Get Instance ID') {
            steps {
                script {
                    echo "Fetching the current instance ID from the Auto Scaling Group..."
                    def instanceId = sh(script: "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ${ASG_NAME} --region ${REGION} --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text", returnStdout: true).trim()
                    
                    echo "Current Instance ID: ${instanceId}"
                    env.INSTANCE_ID = instanceId
                }
            }
        }
        stage('Create AMI') {
            steps {
                script {
                    def timestamp = new Date().format("yyyyMMddHHmmss", TimeZone.getTimeZone('UTC'))
                    def amiName = "${AMI_NAME_PREFIX}-${timestamp}"
                    
                    echo "Creating AMI from the EC2 instance..."
                    def amiId = sh(script: "aws ec2 create-image --instance-id ${INSTANCE_ID} --name ${amiName} --region ${REGION} --output text --query ImageId", returnStdout: true).trim()
                    
                    echo "AMI ID: ${amiId}"
                    echo "Waiting for AMI to become available..."
                    sh "aws ec2 wait image-available --image-ids ${amiId} --region ${REGION}"
                    
                    echo "AMI is now available."
                    env.AMI_ID = amiId
                }
            }
        }
        stage('Create Launch Template Version') {
            steps {
                script {
                    echo "Creating a new Launch Template version with the new AMI..."
                    def createLtVersionCmd = """
                        aws ec2 create-launch-template-version --launch-template-name ${LAUNCH_TEMPLATE_NAME} \
                        --source-version 1 --launch-template-data ImageId=${AMI_ID} --region ${REGION} --query 'LaunchTemplateVersion.VersionNumber' --output text
                    """
                    def latestVersion = sh(script: createLtVersionCmd, returnStdout: true).trim()
                    
                    echo "Latest Launch Template Version: ${latestVersion}"
                    env.LATEST_VERSION = latestVersion
                }
            }
        }
        stage('Update Auto Scaling Group') {
            steps {
                script {
                    echo "Updating the Auto Scaling Group to use the new Launch Template version..."
                    sh "aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${ASG_NAME} --launch-template LaunchTemplateName=${LAUNCH_TEMPLATE_NAME},Version=${LATEST_VERSION} --region ${REGION}"
                }
            }
        }
        stage('Replace Instances') {
            steps {
                script {
                    echo "Replacing instances in the Auto Scaling Group..."
                    sh """
                        aws autoscaling start-instance-refresh --auto-scaling-group-name ${ASG_NAME} \
                        --preferences '{"InstanceWarmup": 300, "MinHealthyPercentage": 50}' --region ${REGION}
                    """
                }
            }
        }
        stage('Terminate Original Instance') {
            steps {
                script {
                    echo "Waiting for the new instance to become healthy..."
                    def healthyInstances = sh(script: "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ${ASG_NAME} --region ${REGION} --query 'AutoScalingGroups[0].Instances[?LifecycleState==`InService`].InstanceId' --output text", returnStdout: true).trim().split()
                    
                    if (healthyInstances.size() > 1) {
                        echo "Terminating the original instance..."
                        sh "aws ec2 terminate-instances --instance-ids ${INSTANCE_ID} --region ${REGION}"
                        
                        echo "Waiting for the original instance to terminate..."
                        sh "aws ec2 wait instance-terminated --instance-ids ${INSTANCE_ID} --region ${REGION}"
                    } else {
                        echo "Only one instance is running, no need to terminate the original instance."
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'Update process completed.'
        }
    }
}
