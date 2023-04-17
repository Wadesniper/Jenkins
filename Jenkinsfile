/* import shared library */
@Library('chocoapp-slack-share-library')_

pipeline {
    // définition des variables d'environnement
    environment {
        IMAGE_NAME = "staticwebsite"
        APP_EXPOSED_PORT = "80"
        IMAGE_TAG = "latest"
        STAGING = "chocoapp-staging"
        PRODUCTION = "chocoapp-prod"
        DOCKERHUB_ID = "choco1992"
        DOCKERHUB_PASSWORD = credentials('dockerhub_password')
        APP_NAME = "WADE"
        STG_API_ENDPOINT = "ip10-0-0-3-cenjc18mjkegg872ced0-1993.direct.docker.labs.eazytraining.fr"
        STG_APP_ENDPOINT = "ip10-0-0-3-cenjc18mjkegg872ced0-8080.direct.docker.labs.eazytraining.fr"
        PROD_API_ENDPOINT = "ip10-0-0-3-cenjc18mjkegg872ced0-1993.direct.docker.labs.eazytraining.fr"
        PROD_APP_ENDPOINT = "ip10-0-0-3-cenjc18mjkegg872ced0-80.direct.docker.labs.eazytraining.fr"
        INTERNAL_PORT = "80"
        EXTERNAL_PORT = "${PORT_EXPOSED}" // 
        CONTAINER_IMAGE = "${DOCKERHUB_ID}/${IMAGE_NAME}:${IMAGE_TAG}"
    }
    agent none
    stages {
       stage('Build image') {
           agent any
           steps {
              script {
                sh 'docker build -t ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG .' // construction de l'image Docker à partir du Dockerfile
              }
           }
       }
       stage('Run container based on builded image') {
          agent any
          steps {
            script {
              sh '''
                  echo "Cleaning existing container if exist"
                  docker ps -a | grep -i $IMAGE_NAME && docker rm -f $IMAGE_NAME // nettoyage des anciens conteneurs s'ils existent
                  docker run --name $IMAGE_NAME -d -p $APP_EXPOSED_PORT:$INTERNAL_PORT  ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG // création d'un nouveau conteneur à partir de l'image construite
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
                   curl -v 172.17.0.1:$APP_EXPOSED_PORT | grep -i "Dimension" // test de l'image en vérifiant la présence de la chaîne "Dimension"
                '''
              }
           }
       }
       stage('Clean container') {
          agent any
          steps {
             script {
               sh '''
                   docker stop $IMAGE_NAME // arrêt du conteneur
                   docker rm $IMAGE_NAME // suppression du conteneur
               '''
             }
          }
      }

      stage ('Login and Push Image on docker hub') {
          agent any
          steps {
             script {
               sh '''
                   echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_ID --password-stdin // connexion à Docker Hub et authentification
                   docker push ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG // envoi de l'image construite sur Docker Hub
               '''
             }
          }
      }

      stage('STAGING - Deploy app') { // Définition d'une étape de déploiement en staging
  agent any // Utilisation d'un agent quelconque pour l'étape

  steps { // Définition des étapes à exécuter pour l'étape
    script { // Utilisation d'un script # Exécution d'une commande shell avec interpolation de variables
      sh """ 
        echo  {\\"your_name\\":\\"${APP_NAME}\\",\\"container_image\\":\\"${CONTAINER_IMAGE}\\", \\"external_port\\":\\"${EXTERNAL_PORT}80\\", \\"internal_port\\":\\"${INTERNAL_PORT}\\"}  > data.json # Écriture d'un fichier JSON contenant des informations sur l'application
        curl -v -X POST http://${STG_API_ENDPOINT}/staging -H 'Content-Type: application/json'  --data-binary @data.json  2>&1 | grep 200 # Envoi des données au endpoint STAGING_API_ENDPOINT
      """
    }
  }
}

stage('PROD - Deploy app') { // Définition d'une étape de déploiement en prod
  when { // Utilisation d'une condition
    expression { GIT_BRANCH == 'origin/main' } // Condition à vérifier
  }
  agent any // Utilisation d'un agent quelconque pour l'étape

  steps { // Définition des étapes à exécuter pour l'étape
    script { 
      sh """ 
        echo  {\\"your_name\\":\\"${APP_NAME}\\",\\"container_image\\":\\"${CONTAINER_IMAGE}\\", \\"external_port\\":\\"${EXTERNAL_PORT}\\", \\"internal_port\\":\\"${INTERNAL_PORT}\\"}  > data.json // Écriture d'un fichier JSON contenant des informations sur l'application
        curl -v -X POST http://${PROD_API_ENDPOINT}/prod -H 'Content-Type: application/json'  --data-binary @data.json  2>&1 | grep 200 # Envoi des données au endpoint PROD_API_ENDPOINT
      """
    }
  }
}

post { // Définition des actions à exécuter après l'exécution de toutes les étapes
  success { // Action à exécuter en cas de succès
    slackSend (color: '#00FF00', message: "Serigne - SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL}) - PROD URL => http://${PROD_APP_ENDPOINT} , STAGING URL => http://${STG_APP_ENDPOINT}") // Envoi d'un message de succès via Slack
  }
  failure { // Action à exécuter en cas d'échec
    slackSend (color: '#FF0000', message: "Oups - FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})") // Envoi d'un message d'échec via Slack
  }   
}  
    }
