pipeline {
    agent any
    environment {
        REGION = 'us-east-1'
        LAUNCH_TEMPLATE_NAME = 'LT-1'
        ASG_NAME = 'Asg-test'
        AMI_NAME_PREFIX = 'test'
    }
    stages {
        stage('Terminate an Instance and Wait for New Instance') {
            steps {
                script {
                    echo "Fetching current instance IDs in the ASG..."
                    def instanceIds = sh(script: "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ${ASG_NAME} --region ${REGION} --query 'AutoScalingGroups[0].Instances[*].InstanceId' --output text", returnStdout: true).trim().split()
                    def instanceIdToTerminate = instanceIds[0]
                    echo "Terminating instance: ${instanceIdToTerminate} to trigger a new instance launch..."

                    sh "aws autoscaling terminate-instance-in-auto-scaling-group --instance-id ${instanceIdToTerminate} --region ${REGION} --no-should-decrement-desired-capacity"
                    
                    echo "Waiting for a new instance to launch..."
                    def newInstanceId = ""
                    retry(5) {
                        sleep(60) // Wait 60 seconds between retries
                        newInstanceId = sh(script: """
                            aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ${ASG_NAME} --region ${REGION} --query 'AutoScalingGroups[0].Instances[?LifecycleState==`InService`].InstanceId' --output text | grep -v ${instanceIdToTerminate}
                        """, returnStdout: true).trim()
                        if (newInstanceId == "") {
                            error "New instance not found yet, retrying..."
                        }
                    }

                    echo "New Instance ID: ${newInstanceId}"
                    sleep(300) // Wait for the instance to be fully initialized
                    env.NEW_INSTANCE_ID = newInstanceId
                }
            }
        }
        stage('Create AMI from New Instance') {
            steps {
                script {
                    echo "Creating a new AMI from the new instance..."
                    def timestamp = new Date().format("yyyyMMddHHmmss", TimeZone.getTimeZone('UTC'))
                    def amiName = "${AMI_NAME_PREFIX}-${timestamp}"
                    def amiId = sh(script: "aws ec2 create-image --instance-id ${NEW_INSTANCE_ID} --name ${amiName} --region ${REGION} --output text --query ImageId", returnStdout: true).trim()
                    sh "aws ec2 wait image-available --image-ids ${amiId} --region ${REGION}"
                    
                    echo "New AMI ID: ${amiId}"
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
                        --source-version 1 --launch-template-data ImageId=${env.AMI_ID} --region ${REGION} --query 'LaunchTemplateVersion.VersionNumber' --output text
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
    }
    post {
        always {
            echo 'Update process completed.'
        }
    }
}
