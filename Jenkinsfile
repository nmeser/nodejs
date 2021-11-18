pipeline {
    agent any
   
    environment {
        PATH=sh(script:"echo $PATH:/usr/local/bin", returnStdout:true).trim()
        AWS_REGION = "us-east-1"
        AWS_ACCOUNT_ID=sh(script:'export PATH="$PATH:/usr/local/bin" && aws sts get-caller-identity --query Account --output text', returnStdout:true).trim()
        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        // ECR_REGISTRY = "0xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com"
        APP_REPO_NAME = "bitay/ci_cd"
        APP_NAME = "Jenkins-test"
        HOME_FOLDER = "/home/ec2-user"
        GIT_FOLDER = sh(script:'echo ${GIT_URL} | sed "s/.*\\///;s/.git$//"', returnStdout:true).trim()
        REPO_NO = "0"
    }
    stages {
        stage('Create ECR Repo') {
            steps {
                
                echo 'Creating ECR Repo for App'
            
                sh """
                   aws ecr create-repository \
                  --repository-name ${APP_REPO_NAME} || true \
                  --image-scanning-configuration scanOnPush=false \
                  --image-tag-mutability MUTABLE \
                  --region ${AWS_REGION}
                """
            }
        }
        stage('Build App Docker Image') {
            steps {
                echo 'Building App Image'
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:latest" .'
                sh 'docker image ls'
            }
        }
        stage('Push Image to ECR Repo') {
            steps {
                echo 'Pushing App Image to ECR Repo'
                sh 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }
        
        stage('Deploy') {
            steps {
                 sh 'docker container ls -a'
                 
              //  sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
              //  sh 'docker pull "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
                sh 'docker container run -p 3000:3000 --name nezihlogin "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
                
            }
        }        
        
        stage('Reports') {
            steps {
		    sh """
		    mkdir -p reports
		    docker cp nezihlogin:/home/BitayTest/Tests/Giris/output.xml reports/.
		    docker cp nezihlogin:/home/BitayTest/Tests/Giris/report.html reports/.
		    docker cp nezihlogin:/home/BitayTest/Tests/Giris/log.html reports/.
		    """
		    
		   script {
		          step(
			            [
			              $class              : 'RobotPublisher',
			              outputPath          : 'reports',
			              outputFileName      : '**/output.xml',
			              reportFileName      : '**/report.html',
			              logFileName         : '**/log.html',
			              disableArchiveOutput: false,
			              passThreshold       : 50,
			              unstableThreshold   : 40,
			              otherFiles          : "**/*.png,**/*.jpg",
			            ]
		          	)
		        }  
	    }
       }
      

        
        
    }
    post {
        always {
	    emailext(attachLog: true, body: 'jenkins-test  maili', subject: 'jenkins-test ', to: 'neser@bitay.com')
            echo 'Deleting all local imagess'
            sh 'docker image prune -af'
            sh 'docker container rm -f nezihlogin'
        }
        failure {

            echo 'Delete the Image Repository on ECR due to the Failure'
            sh """
                aws ecr delete-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --region ${AWS_REGION}\
                  --force
                """
            sh 'docker container rm -f nezihlogin'
           
        }
    }
}
