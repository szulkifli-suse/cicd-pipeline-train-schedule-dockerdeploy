pipeline {
    agent any
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
                    app = docker.build("darkwunan/train-schedule")
                    app.inside {
                        sh 'echo $(curl localhost:8080)'
                    }
                }
            }
        }
        stage('Push Docker Image to Git') {
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
        stage('Approval to Deploy for Staging Environment') {
            when {
                branch 'master'
            }
            steps {
               input 'Deploy to Staging Environment?'
               milestone(1)
            }
        
        }
        stage('DeployToStaging') {
            when {
                branch 'master'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@122.248.200.22 \"docker pull darkwunan/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@122.248.200.22 \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@122.248.200.22 \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@122.248.200.22 \"docker run --restart always --name train-schedule -p 8080:8080 -d darkwunan/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
         stage('Push to GitOps for Rancher') {
            when {
                branch 'master'
            }
            steps {
            sh """
            git clone https://github.com/szulkifli-suse/fleet-examples.git --branch=master 
            cd fleet-examples/simple/
            pwd
            sed -i "s/train-schedule:12/train-schedule:${env.BUILD_NUMBER}/g" frontend-deployment.yaml
            cat frontend-deployment.yaml
            git add .
            git commit -m "Commit new changes"
            git remote set-url origin https://github.com/szulkifli-suse/fleet-examples.git
            git push origin master
            """
            }
        }
    }
}
