
pipeline {
    agent none
    environment {
        DOCKERHUB_AUTH = credentials('dockerhub')
        ID_DOCKER = "${DOCKERHUB_AUTH_USR}"
        PORT_EXPOSED = "80"
        IMAGE_NAME = "alpinebootcamp26"
        IMAGE_TAG = "v1.3"
        DOCKER_USERNAME = 'dockerhub'
    }
    stages {
      stage ('Build image'){
          agent any
          steps {
            script {
                sh 'docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }
      }
      stage('Run container based on builded image and test') {
        agent any
        steps {
         script {
           sh '''
              echo "Clean Environment"
              docker rm -f $IMAGE_NAME || echo "container does not exist"
              docker run --name $IMAGE_NAME -d -p ${PORT_EXPOSED}:5000 -e PORT=5000 ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
              sleep 5
              curl http://172.17.0.1:${PORT_EXPOSED} | grep -q "Hello world!"
           '''
         }
        }
      }
      stage('Clean Container'){
          agent any
          steps {
              script {
                  sh '''
                      docker stop $IMAGE_NAME
                      docker rm $IMAGE_NAME
                  '''
              }
          }
      }
      stage('Login and Push Image on docker hub'){
          agent any
          steps {
              script {
                  sh '''
                    docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW
                    docker push ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
                  '''
              }
          }
      }

      stage('Deploy in staging'){
          agent any
            environment {
                SERVER_IP = "54.227.172.12"
            }
          steps {
            sshagent(['SSH_AUTH_SERVER']) {
                sh '''
                    ssh -o StrictHostKeyChecking=no -l ubuntu $SERVER_IP "docker rm -f $IMAGE_NAME || echo 'All deleted'"
                    ssh -o StrictHostKeyChecking=no -l ubuntu $SERVER_IP "docker pull $DOCKER_USERNAME/$IMAGE_NAME:$IMAGE_TAG || echo 'Image Download successfully'"
                    sleep 30
                    ssh -o StrictHostKeyChecking=no -l ubuntu $SERVER_IP "docker run --rm -dp $PORT_EXPOSED:5000 -e PORT=5000 --name $IMAGE_NAME $DOCKER_USERNAME/$IMAGE_NAME:$IMAGE_TAG"
                    sleep 5
                    curl -I http://$SERVER_IP:$PORT_EXPOSED
                '''
            }
          }
      }
      stage('Deploy in prod'){
          agent any
            environment {
                HOSTNAME_DEPLOY_PROD = "54.175.31.22"
            }
          steps {
            sshagent(credentials: ['SSH_AUTH_SERVER']) {
                sh '''
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    ssh-keyscan -t rsa,dsa ${HOSTNAME_DEPLOY_PROD} >> ~/.ssh/known_hosts
                    command1="docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW"
                    command2="docker pull $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                    command3="docker rm -f alpinebootcampp || echo 'app does not exist'"
                    command4="docker run -d -p 80:5000 -e PORT=5000 --name alpinebootcampp $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                    ssh -t ubuntu@${HOSTNAME_DEPLOY_PROD} \
                        -o SendEnv=IMAGE_NAME \
                        -o SendEnv=IMAGE_TAG \
                        -o SendEnv=DOCKERHUB_AUTH_USR \
                        -o SendEnv=DOCKERHUB_AUTH_PSW \
                        -C "$command1 && $command2 && $command3 && $command4"
                '''
            }
          }
      }        

    }
}
