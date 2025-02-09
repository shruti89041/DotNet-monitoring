pipeline{
    agent any
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout From Git'){
            steps{
                git branch: 'main', url: 'https://github.com/shruti89041/DotNet-monitoring.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Dotnet-Webapp \
                    -Dsonar.projectKey=Dotnet-Webapp '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage("TRIVY File scan"){
            steps{
                sh "trivy fs . > trivy-fs_report.txt" 
            }
        }
        stage("OWASP Dependency Check"){
            steps{
                dependencyCheck additionalArguments: '--scan ./ --format XML ', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage("Docker Build & tag"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "export IMAGE_TAG=${env.BUILD_NUMBER} && make image"
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image shrutifarkya/dotnet-monitoring:${env.BUILD_NUMBER} > trivy.txt"
            }
        }
        stage("Docker Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "make push"
                    }
                }
            }
        }
        stage("Deploy to container"){
            steps{
                sh "docker run -d --name dotnet -p 5000:5000 shrutifarkya/dotnet-monitoring:${env.BUILD_NUMBER}"
            } 
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh "kubectl run dotnet-monitoring --image=shrutifarkya/dotnet-monitoring:${env.BUILD_NUMBER} --image-pull-policy=Always"
                    sh "kubectl expose pod dotnet-monitoring --port=5000 --target-port=5000 --name dotnet-monitoring-service --type=NodePort"
                    sh "kubectl create ingress dotnet-monitoring-ingress --rule=dotnet-monitoring.example.com/*=dotnet-monitoring-service:5000 --class=nginx"
                }
             }
         }

    }
}
