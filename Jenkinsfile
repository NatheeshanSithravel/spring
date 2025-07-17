pipeline {
  environment {
     ENV="stg"   //Change the environment accordingly ex: stg for staging and  pr for production
     PROJECT = "Natheeshan"
     APP_NAME = "spring"      //Change the application name , which will also be the deployment name
    // CIR = "${ENV}-docker-reg.mobitel.lk"
     CIR_USER = 'natheeshan'
     CIR_PW = 'Qwerty@123'
     KUB_NAMESPACE = "db"               //Change the namespace accordingly
     IMAGE_TAG = "natheeshan/${APP_NAME}:${ENV}"
     EXPOSE_PORT="8080"                    //Change the service expose port accordingly
  //   HARBOUR_SECRET="harbor-intsys"              //Change the harbour secret name accordingly
     
    }
    agent none 
    stages {  

       /*
        stage('Run SonarQube analysis') {
            agent any
            steps {
                script {
                    def scannerHome = tool 'sonarscanner'
                    withSonarQubeEnv('sonarserver') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=mSMS_performance_monitor_backend -Dsonar.projectName='mSMS_performance_monitor_backend'"
                    }
                }
            }
        }
		 */
      
      stage('Build & test') {
        agent {
              docker  {
           		image 'maven:3.9.6-amazoncorretto-21'
           		args '-v /root/.m2:/root/.m2 --user root'
                }
        }
        steps {
            sh "mvn -Dmaven.test.skip=true clean install -X"
        }
      }  
	  stage('Building & Deploy Image') {
      	agent any
		    steps{
              sh '''
              
          		docker login -u ${CIR_USER} -p ${CIR_PW} ${CIR}
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
                docker  {
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
      
      stage ('Remove local Image'){
      agent any
           steps {
                sh 'docker image rm ${IMAGE_TAG}'
            }
      
      }
     
        stage('Deploy cluster') {
              agent {
                 docker  {
                       //image "${ENV}-docker-reg.mobitel.lk/mobitel_pipeline/cicdtools:1"
                   	   image 'inovadockerimages/cicdtools:latest' 
                         args '-v /root/.cert:/root/.cert --user root'   
                        }
                    }
             steps {
               
               sh '''
               
               mkdir -p /root/.kube/
               cp /home/rancher/.kube/config /root/.kube/
               '''
               script {
               def isDeployed = sh(returnStatus: true, script: 'kubectl -n ${KUB_NAMESPACE} set image deployment/${APP_NAME}  ${APP_NAME}=${IMAGE_TAG}  --record ')
                if (isDeployed != 0) {
                        sh '''
                        kubectl -n ${KUB_NAMESPACE} create deployment ${APP_NAME}  --image=${IMAGE_TAG} 
               			kubectl -n ${KUB_NAMESPACE} expose deployment ${APP_NAME}  --name=${APP_NAME} --port=${EXPOSE_PORT}
                        
                        ## Replace the harbour image policy secret name
			   			kubectl -n ${KUB_NAMESPACE} patch deployment ${APP_NAME} --patch \'{"spec": {"template": {"spec": {"imagePullSecrets": [{"name": "'"${HARBOUR_SECRET}"'" }]}}}}\'
                        
                        ## Set resource limits
            			kubectl -n ${KUB_NAMESPACE} patch deployment ${APP_NAME} --type=\'json\' -p=\'[{"op": "add","path": "/spec/template/spec/containers/0/resources","value": {"limits": {"memory": "512Mi"}}}]\'
						
						## Replace the deployment name
					    ##kubectl -n ${KUB_NAMESPACE} patch deployment $deploy -p \'{"spec":{"template":{"spec":{"containers":[{"name": "'"${APP_NAME}"'","imagePullPolicy":"IfNotPresent"}]}}}}\'
                        
                        '''
                    }
               }              
            }
          }      
      
      
        }   
}
