pipeline {
    agent any
    stages {
        stage('Pull Latest Code') {
            steps {
                sh '''
                # Switch to root user
                #sudo -i
                
                # Navigate to the Jenkins directory
                cd /var/www/html/jenkins
                
                # Pull the latest changes from the repository
                git pull https://github.com/salmanlarijanbask/jenkins.git
                '''
            }
        }
    }
}
