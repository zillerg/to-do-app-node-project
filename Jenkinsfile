pipeline{
    agent any
    tools {
        terraform 'tf'
    }
    environment {
        IMAGE_NAME = "zlkr/to-do-app:node-alpine-${BUILD_NUMBER}"
    }
    stages {
        
        stage("Run app on Docker"){
            agent{
                docker{
                    image 'node:12-alpine'
                }
            }
            steps{
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'yarn install --production'
                    sh 'npm install'
                }   
            }
        } 
        
        stage('build image') {
            steps {
                script {
                    echo "building the docker image"
                     withCredentials([usernamePassword(credentialsId: 'docker-hub', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh 'docker build -t ${IMAGE_NAME} .'
                        sh 'docker image ls'
                    echo "push Image to DockerHub"
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                        sh 'docker push ${IMAGE_NAME}'
                    }
                }
            }
        }
        
        stage('provision server') {
           steps {
                script {
                    dir('Terraform') {
                        sh "terraform init"
                        sh "terraform apply --auto-approve"
                        EC2_PUBLIC_IP = sh(script: "terraform output ec2_public_ip",returnStdout: true).trim()
                    }
                }
            }
        }

        stage('deploy') {
            environment {
                DOCKER_CREDS = credentials('docker-hub')
            }

            steps {
                script {
                   echo "waiting for EC2 server to initialize" 
                   sleep(time: 90, unit: "SECONDS") 
                   echo 'deploying docker image to EC2...'
                   echo "${EC2_PUBLIC_IP}"
                   /* def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME}"
                   sh "echo 'IMAGE=${IMAGE_NAME}' > .env" */
                   def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
                   def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"

            
                    sshagent(['firstkey']) {
    /*                  sh "scp -o StrictHostKeyChecking=no ./.env ec2-user@${EC2_PUBLIC_IP}:/home/ec2-user"  
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ec2-user@${EC2_PUBLIC_IP}:/home/ec2-user"
                        sh 'ssh -o StrictHostKeyChecking=no ec2-user@${EC2_PUBLIC_IP} ${shellCmd}' 
                        sh "ssh -o StrictHostKeyChecking=no ec2-user@${EC2_PUBLIC_IP} echo $PASS | docker login -u $USER --password-stdin && docker-compose -f docker-compose.yaml up --detach"
                        
    */ 
                        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
                        sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"


                    }
                   
                }
            }
        }
    }
    post { 
        success { 
            sleep(time: 100, unit: "SECONDS")
            sh "cd Terraform && terraform destroy --auto-approve"
        }
    }
  
}
