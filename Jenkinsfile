pipeline { 
  environment {
     ENV="stg"
     PROJECT = "spring-sample"
     APP_NAME = "spring-sample"
     CIR_USER = 'natheeshan'
     CIR_PW = 'Qwerty@123'
     KUB_NAMESPACE = "Natheeshan"
     IMAGE_TAG = "natheeshan/${APP_NAME}:${ENV}.${env.BUILD_NUMBER}"
     EXPOSE_PORT="4040"
     HARBOUR_SECRET="harbor-intsys"
  }

  agent none

  stages {  

    stage('Run SonarQube analysis') {
      agent any
      steps {
        script {
          def scannerHome = tool 'sonar-scanner'
          withSonarQubeEnv('sonar-server') {
            sh "${scannerHome}/bin/sonar-scanner -Dsonar.sources=./src -Dsonar.java.binaries=target/classes -Dsonar.projectKey=${PROJECT} -Dsonar.projectName=${PROJECT}"
          }
        }
      }
    }

    stage('Build & test') {
      agent {
        docker {
          image 'maven:3.9.6-amazoncorretto-21'
          args '-v /root/.m2:$JENKINS_HOME/.m2 --user root'
        }
      }
      steps {
        echo "$JENKINS_HOME"
        sh "mvn -Dmaven.test.skip=true clean install -X"
      }
    }

    stage('Building & Deploy Image') {
      agent any
      steps {
        sh '''
          docker login -u ${CIR_USER} -p ${CIR_PW} 
          mkdir -p dockerImage/
          cp Dockerfile dockerImage/
          cp target/*.jar dockerImage/
          docker build --tag=${IMAGE_TAG} dockerImage/.
          docker push ${IMAGE_TAG}
        '''
      }
    }

    stage('Trivy-Scan') {
      agent {
        docker {
          image 'aquasec/trivy:latest'
          args '--entrypoint="" -v /var/jenkins_home/trivy-reports:/reports -v trivy-cache:/root/.cache/ --user root'
        }
      }
      steps {
        script {
          sh "trivy image --no-progress  --timeout 15m -f table  ${IMAGE_TAG}"
        }
      }
    }

    /*
    stage ('Remove local Image'){
      agent any
      steps {
        sh 'docker image rm ${IMAGE_TAG}'
      }
    }
    */

    stage('Deploy cluster') {
      agent any
      steps {
        sh '''
          mkdir -p /root/.kube/
          cp /home/rancher/.kube/config /root/.kube/
        ''' 
        script {
          def isDeployed = sh(returnStatus: true, script: 'kubectl -n ${KUB_NAMESPACE} set image deployment/${APP_NAME} ${APP_NAME}=${IMAGE_TAG} --record')
          if (isDeployed != 0) { 
            sh '''
              minikube kubectl -- create deployment ${APP_NAME} --image=${IMAGE_TAG} 
              minikube kubectl -- expose deployment ${APP_NAME} --name=${APP_NAME} --type=NodePort --port=${EXPOSE_PORT}
              minikube kubectl -- patch deployment ${APP_NAME} --type='json' -p='[{"op": "add","path": "/spec/template/spec/containers/0/resources","value": {"limits": {"memory": "512Mi"}}}]'
            '''
          }
        }
      }
    }

  } // <-- closes "stages"
} // <-- closes "pipeline"
