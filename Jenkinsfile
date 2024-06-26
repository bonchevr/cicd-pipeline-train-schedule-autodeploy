pipeline {
    agent any
    environment {
        DOCKER_IMAGE_NAME = "bonchevr/train-schedule"
        CANARY_REPLICAS = 0
        KUBE_CONFIG = credentials('kubeconfig') // Assuming you've set up Kubernetes credentials in Jenkins
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 1
            }
            steps {
                script {
                    sh "kubectl apply -f train-schedule-kube-canary.yml --kubeconfig=${KUBE_CONFIG}"
                }
            }
        }
        stage('SmokeTest') {
            when {
                branch 'master'
            }
            steps {
                script {
                    sleep(time: 5)
                    def response = httpRequest(
                        url: "http://$KUBE_MASTER_IP/",
                        timeout: 30
                    )
                    if (response.status != 200) {
                        error("Smoke test against canary deployment failed.")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                milestone(1)
                script {
                    sh "kubectl apply -f train-schedule-kube.yml --kubeconfig=${KUBE_CONFIG}"
                }
            }
        }
    }
    post {
        cleanup {
            script {
                sh "kubectl delete -f train-schedule-kube-canary.yml --kubeconfig=${KUBE_CONFIG}"
            }
        }
    }
}
