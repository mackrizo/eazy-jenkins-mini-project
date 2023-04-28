/* import shared library */
library('shared-library')

pipeline {
    environment {
        IMAGE_NAME = "staticwebsite"
        IMAGE_TAG = "latest"
        STAGING = "mackrizo-staging"
        PRODUCTION = "mackrizo-production"
        APP_CONTAINER_PORT = "5000"
        APP_EXPOSED_PORT = "8080"
        DOCKERHUB_ID = "mackrizo"
        DOCKERHUB_MACKRIZO = credentials('dockerhub_mackrizo')
    }
    agent none
    stages {
       stage('Build image') {
           agent any
           steps {
              script {
                sh 'docker build -t ${DOCKERHUB_ID}/${IMAGE_NAME}:${IMAGE_TAG} .'
              }
           }
       }
       stage('Run container based on builded image') {
          agent any
          steps {
            script {
              sh '''
                  echo "Cleaning existing container if exist"
                  docker ps -a | grep -i ${IMAGE_NAME} && docker rm -f ${IMAGE_NAME}
                  docker run --name ${IMAGE_NAME} -d -p ${APP_CONTAINER_PORT}:${APP_EXPOSED_PORT} -e PORT=${APP_EXPOSED_PORT} ${DOCKERHUB_ID}/${IMAGE_NAME}:${IMAGE_TAG}
                  sleep 5
              '''
             }
          }
       }
       stage('Test image') {
           agent any
           steps {
              script {
                sh '''
                   curl http://172.17.0.1 | grep -q "Mongongu"
                '''
              }
           }
       }
       stage('Clean container') {
          agent any
          steps {
             script {
               sh '''
                   docker stop ${IMAGE_NAME}
                   docker rm ${IMAGE_NAME}
               '''
             }
          }
      }

      stage ('Login and Push Image on docker hub') {
          agent any
          steps {
             script {
               sh '''
                   echo ${DOCKERHUB_PASSWORD} | docker login -u ${DOCKERHUB_ID} --password-stdin
                   docker push ${DOCKERHUB_ID}/${IMAGE_NAME}:${IMAGE_TAG}
               '''
             }
          }
      }

      stage('Push image in staging and deploy it') {
        when {
            expression { GIT_BRANCH == 'origin/main' }
            }
        agent any
        environment {
            HEROKU_API_KEY = credentials('heroku_api_key')
        }
        steps {
           script {
             sh '''
                apk --no-cache add npm
                npm install -g heroku
                heroku container:login
                heroku create ${STAGING} || echo "projet already exist"
                heroku container:push -a ${STAGING} web
                heroku container:release -a ${STAGING} web
             '''
           }
        }
     }
     stage('Push image in production and deploy it') {
       when {
           expression { GIT_BRANCH == 'origin/main' }
            }
       agent any
       environment {
           HEROKU_API_KEY = credentials('heroku_api_key')
       }
       steps {
          script {
            sh '''
               apk --no-cache add npm
               npm install -g heroku
               heroku container:login
               heroku create ${PRODUCTION} || echo "projets already exist"
               heroku container:push -a ${PRODUCTION} web
               heroku container:release -a ${PRODUCTION} web
            '''
          }
       }
     }
  }
  post {
     always {
       script {
         slackNotifier currentBuild.result
     }
    }
  }
}
