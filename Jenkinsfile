pipeline {
  agent any

  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "kilian/devops-app:${GIT_COMMIT}"
    applicationURL="http://mytpm.eastus.cloudapp.azure.com"
    applicationURI="/increment/99"
      }

  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded 
            }
        }   
 
      stage('UNIT test & jacoco ') {
          steps {
            sh "mvn test"
          }
          post {
            always {
              junit 'target/surefire-reports/*.xml'
              jacoco execPattern: 'target/jacoco.exec'
            }
          }
    
        }

      stage('Mutation Tests - PIT') {
  	    steps {
    	    sh "mvn org.pitest:pitest-maven:mutationCoverage"
  	      }
    	  post {
     	    always {
       	    pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
     	     }
      	  }
	    }

      stage('Vulnerability Scan - Docker Trivy') {
   	    steps {
        	withCredentials([string(credentialsId: 'privekilian', variable: 'TOKEN')]) {
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
           sh "sed -i 's#token_github#${TOKEN}#g' trivy-image-scan.sh" 	 
            sh "sudo bash trivy-image-scan.sh"}
       	}
   	}
 	}

      stage('Docker Build and Push') {
  	    steps {
    	    withCredentials([string(credentialsId: 'docker_hub_kilian', variable: 'DOCKER_HUB_PASSWORD')]) {
      	    sh 'sudo docker login -u gkill -p $DOCKER_HUB_PASSWORD'
      	    sh 'printenv'
      	    sh 'sudo docker build -t gkill/devops-app:""$GIT_COMMIT"" .'
      	    sh 'sudo docker push gkill/devops-app:""$GIT_COMMIT""'
    	}

  	}
	}

      stage('Deployment Kubernetes  ') {
  	    steps {
    	    withKubeConfig([credentialsId: 'kubeconfig']) {
           	sh "sed -i 's#replace#gkill/devops-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
           	sh "kubectl apply -f k8s_deployment_service.yaml"
         	}
  	}
    }

      stage('SonarQube - SAST') {
     	 
       	steps {
   		catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
     	withSonarQubeEnv('SonarQubeConfig') {


        	sh "mvn sonar:sonar \
  -Dsonar.projectKey=projectkilian \
  -Dsonar.projectName=projectkilian \
  -Dsonar.host.url=http://mytpm.eastus.cloudapp.azure.com:9999"
         }
 
   		}
   	}
     	 
 
 	}

      stage('Vulnerability Scan - Docker') {
        steps {
    	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
			      sh "mvn dependency-check:check"
    	    }
   	 }
   	    post {
  	      always {
   			    dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
   			 }
   	 }
 }

    }
  }
