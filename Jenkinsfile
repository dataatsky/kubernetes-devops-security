pipeline {
  agent any

  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "siddharth67/numeric-app:${GIT_COMMIT}"
    applicationURL = "http://localhost/"
    applicationURI = "/increment/99"
  }

  stages {

    stage('Build Artifact - Maven') {
      steps {
        sh "mvn clean package -DskipTests=true"
        archive 'target/*.jar'
      }
    }

    stage('Unit Tests - JUnit and Jacoco') {
      steps {
        sh "mvn test"
      }
    }
    
    stage('Mutation Tests - PIT') {
      steps {
        sh "mvn org.pitest:pitest-maven:mutationCoverage"
      }
    }

    stage('SonarQube - SAST') {
      steps {
        withSonarQubeEnv('SonarQube') {
        sh "mvn sonar:sonar -Dsonar.projectKey=jenkins -Dsonar.host.url=http://localhost:9000"
      }
       timeout(time: 2, unit: 'MINUTES') {
         script {
           waitForQualityGate abortPipeline: true
          }
        }
      }
    }
    
    stage('Vulnerability Scan - Docker') {
      steps {
        parallel(
          "Dependency Scan": {
            sh "mvn dependency-check:check"
          },
          "Trivy Scan": {
            sh "bash trivy-docker-image-scan.sh"
          },
          "OPA Conftest": {
            sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
          }
        )
      }
    }

    stage('Docker Build and Push') {
      steps {
        withDockerRegistry([credentialsId: "dockerhub", url: ""]) {
          sh 'printenv'
          sh 'sudo docker build -t asdrt4fd/test:""$GIT_COMMIT"" .'
          sh 'docker push asdrt4fd/test:""$GIT_COMMIT""'
        }
      }
    }

    stage('Vulnerability Scan - Kubernetes') {
      steps {
        sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
      }
    }

    // stage('Kubernetes Deployment - DEV') {
    //   steps {
    //     withKubeConfig([credentialsId: 'kubeconfig']) {
    //       sh "sed -i 's#replace#siddharth67/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
    //       sh "kubectl apply -f k8s_deployment_service.yaml"
    //     }
    //   }
    // }

    stage('K8S Deployment - DEV') {
      steps {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment.sh"
            }
      }
    }

    stage('testkub') {
      steps {
        {
          withKubeConfig([credentialsId: 'kubeconfig']) {
          sh "bash k8s-deployment-rollout-status.sh"
          }
        }
      }
    }

  }  


  post {
    always {
      junit 'target/surefire-reports/*.xml'
      jacoco execPattern: 'target/jacoco.exec'
      pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
      dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
    }

    // success {

    // }

    // failure {

    // }
  }
}

