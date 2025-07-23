pipeline { 
  environment {
    ENV = "stg"   
    PROJECT = "APP-NEW"
    APP_NAME = "APP"
    CIR_USER = 'natheeshan'
    CIR_PW = 'Qwerty@123'
    KUB_NAMESPACE = "default"
    IMAGE_TAG = "natheeshan/${APP_NAME}:${ENV}.${env.BUILD_NUMBER}"
    EXPOSE_PORT = "4040"
    HARBOUR_SECRET = "harbor-intsys"
  }

  agent none

  stages {
    stage('Build & Test') {
      agent {
        docker {
          image 'maven:3.9.6-amazoncorretto-21'
          args '-v /root/.m2:$JENKINS_HOME/.m2 --user root'
        }
      }
      steps {
        echo "$JENKINS_HOME"
        sh "mvn -Dmaven.test.skip=true clean install -X"
        sh "pwd && ls -l"
        sh "mvn dependency:copy-dependencies -DoutputDirectory=target/dependency"
        sh "echo 'Listing copied dependencies:' && ls -l target/dependency"
      }
    }

    stage('Run SonarQube Analysis') {
      agent any
      steps {
        script {
          def scannerHome = tool 'sonar-scanner'
          withSonarQubeEnv('sonar-server') {
            // Use comma-separated absolute paths if glob fails
            def libs = sh(script: "find target/dependency -name '*.jar' | tr '\\n' ',' | sed 's/,\$//'", returnStdout: true).trim()

            if (!libs) {
              error("No JAR files found in target/dependency for sonar.java.libraries!")
            }

            sh """
              ${scannerHome}/bin/sonar-scanner \
                -Dsonar.sources=./src \
                -Dsonar.java.binaries=target/classes \
                -Dsonar.projectKey=${APP_NAME} \
                -Dsonar.projectName=${PROJECT} \
                -Dsonar.dependencyCheck.reportPath=dependency-check-report/dependency-check-report.xml \
                -Dsonar.java.libraries=${libs}
            """
          }
        }
      }
    }
  }
}
