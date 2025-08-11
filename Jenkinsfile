pipeline {
  environment {
    ID_DOCKER   = "${ID_DOCKER_PARAMS}"
    IMAGE_NAME  = "alpinehelloworld"
    IMAGE_TAG   = "latest"
    // PORT_EXPOSED doit être défini en paramètre de job
    STAGING     = "${ID_DOCKER}-staging"
    PRODUCTION  = "${ID_DOCKER}-production"
  }

  agent none

  stages {

    stage('Build image') {
      agent any
      steps {
        script {
          sh 'DOCKER_BUILDKIT=1 docker build -t ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG .'
        }
      }
    }

    stage('Run container based on builded image') {
      agent any
      steps {
        script {
          sh '''
            echo "Clean Environment"
            docker rm -f $IMAGE_NAME || echo "container does not exist"
            docker run --name $IMAGE_NAME -d -p ${PORT_EXPOSED}:5000 -e PORT=5000 ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
          '''
        }
      }
    }

    stage('Test image') {
      agent any
      steps {
        script {
          sh '''
            # Attendre que l’appli réponde puis vérifier le contenu
            for i in $(seq 1 20); do
              if curl -fsS http://172.17.0.1:${PORT_EXPOSED} | grep -q "Hello world!"; then
                exit 0
              fi
              sleep 1
            done
            echo "L’application ne répond pas comme attendu"
            exit 1
          '''
        }
      }
    }

    stage('Clean Container') {
      agent any
      steps {
        script {
          sh '''
            docker stop $IMAGE_NAME || true
            docker rm $IMAGE_NAME || true
          '''
        }
      }
    }

    stage('Login and Push Image on docker hub') {
      agent any
      environment {
        DOCKERHUB_PASSWORD = credentials('dockerhub')
      }
      steps {
        script {
          sh '''
            echo $DOCKERHUB_PASSWORD_PSW | docker login -u $ID_DOCKER --password-stdin
            docker push ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
          '''
        }
      }
    }

    stage('Push image in staging and deploy it') {
      when { expression { GIT_BRANCH == 'origin/master' } }
      agent any
      environment {
        HEROKU_API_KEY = credentials('heroku_api_key')
      }
      steps {
        script {
          // Utiliser l’image officielle Heroku CLI, et monter le socket Docker
          docker.image('heroku/cli').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
            sh '''
              # Auth Heroku Container Registry
              heroku container:login

              # Créer l’app si besoin
              heroku apps:info -a $STAGING >/dev/null 2>&1 || heroku create $STAGING

              # Re-tag vers le registre Heroku et push
              docker tag ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG registry.heroku.com/$STAGING/web
              docker push registry.heroku.com/$STAGING/web

              # Release
              heroku container:release web --app $STAGING
            '''
          }
        }
      }
    }

    stage('Push image in production and deploy it') {
      when { expression { GIT_BRANCH == 'origin/production' } }
      agent any
      environment {
        HEROKU_API_KEY = credentials('heroku_api_key')
      }
      steps {
        script {
          docker.image('heroku/cli').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
            sh '''
              heroku container:login
              heroku apps:info -a $PRODUCTION >/dev/null 2>&1 || heroku create $PRODUCTION
              docker tag ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG registry.heroku.com/$PRODUCTION/web
              docker push registry.heroku.com/$PRODUCTION/web
              heroku container:release web --app $PRODUCTION
            '''
          }
        }
      }
    }

  }
}
