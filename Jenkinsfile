pipeline {
    agent any
	environment{
	    ENVIRONMENT = 'PROD'
	    TIME_STAMP = new java.text.SimpleDateFormat('yyyy_MM_dd').format(new Date())
            DOCKER_TAG = "${ENVIRONMENT}_${TIME_STAMP}_${env.BUILD_NUMBER}"
    }
	
 stages {
      stage('checkout') {
           steps {
                git credentialsId: 'github-creds', url: 'https://github.com/kishanth94/Java_Containerized_Application_CICD.git' 
          }
        }
		
	 stage('Quality Gate Status Check'){
            steps{
                script{
			withSonarQubeEnv(credentialsId: 'sonar-token') { 
                        // Get Home Path of Maven 
                        def mvnHome = tool name: 'maven-3', type: 'maven'
			sh "${mvnHome}/bin/mvn clean sonar:sonar"
                       	  }
			            timeout(time: 20, unit: 'SECONDS') {
			            def qg = waitForQualityGate()
				                if (qg.status != 'OK') {
					                 error "Pipeline aborted due to quality gate failure: ${qg.status}"
				                }
                          }
                }
            }  
        }
		
	 stage("Maven Build"){
            steps{
                script{
                // Get Home Path of Maven 
                def mvnHome = tool name: 'maven-3', type: 'maven'
                sh "${mvnHome}/bin/mvn package"
                }
            }
        }
		
    stage("Upload War To Nexus"){
	    steps{
		script{
		    def mavenPom = readMavenPom file: 'pom.xml'
		    def nexusRepoName = mavenPom.version.endsWith("SNAPSHOT") ? "LoginWebApp-snapshot" : "LoginWebApp-release"
		    nexusArtifactUploader artifacts: [
			[
			    artifactId: 'LoginWebApp', 
		        classifier: '', 
			    file: "target/LoginWebApp-${mavenPom.version}.war", 
			    type: 'war'
			]
			], 
			    credentialsId: 'nexus3', 
			    groupId: 'com.devops4solutions', 
			    nexusUrl: '172.31.4.60:8081', 
			    nexusVersion: 'nexus3', 
			    protocol: 'http', 
			    repository: nexusRepoName, 
			    version: "${mavenPom.version}"    
           }
		}
	}
	
    stage('Docker Build and Tag') {
           steps {
                sh '''
		      echo $WORKSPACE
		      mv target/*.war target/LoginWebApp.war
		      source ~/.profile
		      echo "$password" | sudo -kS docker build -t samplewebapp:${DOCKER_TAG} .
                      echo "$password" | sudo -kS docker tag samplewebapp:${DOCKER_TAG} kishanth1994/samplewebapp:${DOCKER_TAG}
		      echo "$password" | sudo -kS docker login --username kishanth1994 --password welcome@123
		   '''
          }
        }
     
    stage('Publish image to Docker Hub') {
            steps {
              script{ 
                sh '''
		     source ~/.profile
		     echo "$password" | sudo -kS docker push kishanth1994/samplewebapp:${DOCKER_TAG}
	             echo "$password" | sudo -kS docker rmi -f kishanth1994/samplewebapp:${DOCKER_TAG}
	             echo "$password" | sudo -kS docker rmi -f samplewebapp:${DOCKER_TAG}
		   '''
            }        
         }
      }
     
	 stage("Copying the playbook and kubernetes manifests to Ansible server"){
		    steps{
			    script{
				    echo "${WORKSPACE}"
					echo "${DOCKER_TAG}"
					        sh '''
					         source ~/.profile
						 echo "$password" | sudo -kS grep -irl {DOCKER_TAG} ${WORKSPACE}/kubernetes/manifests-yamls/deployment.yaml | xargs sed -i "s/{DOCKER_TAG}/${DOCKER_TAG}/g"
						 echo "$password" | sudo -kS scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/id_rsa ${WORKSPACE}/kubernetes/manifests-yamls/*.yaml ansadmin@172.31.26.189:/etc/ansible/kubernetes/
							'''     
				}
            }
        }
		
      stage('manual approval'){
            steps{
                script{
                    timeout(10) {
                        mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Go to build url and approve the deployment request <br> URL of the build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "devopsawsfreetier@gmail.com";  
                        input(id: "Deploy Gate", message: "Deploy ${params.project_name}?", ok: 'Deploy')
                    }
                }
            }
        }
	
	 stage('Deploying application on k8s cluster from ansible-server using playbook') {
            steps {
               script{
			        sh 'ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/id_rsa ansadmin@172.31.26.189 "/home/ansadmin/bin/kubectl config get-contexts; /home/ansadmin/bin/kubectl config use-context arn:aws:eks:eu-west-1:999474717263:cluster/eks-demo-cluster; whoami && hostname; ansible-playbook /etc/ansible/kubernetes/playbook-deployment-service.yaml; sudo rm -rf /etc/ansible/kubernetes/*.yaml;"'   
               }
            }
        }
    }
	
	post {
	  always {
	    echo 'Deleting the Workspace'
	    deleteDir() /* Clean Up our Workspace */
	  }
	    success {
		mail to: 'devopsawsfreetier@gmail.com',
		  subject: "Success Build Pipeline: ${currentBuild.fullDisplayName}",
		  body: "The pipeline ${env.BUILD_URL} completed successfully"
	    }
	    failure {
  	        mail to: 'devopsawsfreetier@gmail.com',
 		  subject: "Failed Build Pipeline: ${currentBuild.fullDisplayName}",
 		  body: "Something is wrong with ${env.BUILD_URL}"
 	    }
    }
}
    
