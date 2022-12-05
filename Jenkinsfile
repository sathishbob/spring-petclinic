pipeline {

	agent none  

    stages {

  	stage('Maven Install') {

    	agent {

      	docker {

        	image 'maven:3.6.3'

        }

      }

      steps {

      	sh 'mvn clean install'

      }

    }

    stage('Docker Build') {

    	agent any

      steps {
        sh 'pwd'
        sh 'ls -l'
      	sh 'docker build -t sathishbob/spring-petclinic:latest .'

      }

    }

    stage('Docker Push') {

    	agent any

      steps {

      	withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {

        	sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPassword}"

          sh 'docker push sathishbob/spring-petclinic:latest'

        }

      }

    }

  }

}
