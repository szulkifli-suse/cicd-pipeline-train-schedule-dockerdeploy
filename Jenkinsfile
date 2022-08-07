pipeline {
    agent any
    environment {
        GIT = credentials('rancher')
    }
    stages {
        stage('Build Java App') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Create Docker Image') {
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
        stage('Push Docker Image to Repository') {
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
        stage('Neuvector Scan image') {
            when {
                branch 'master'
            }
        steps {  
        neuvector egistrySelection: 'darkwunan', repository: 'darkwunan/train-schedule', scanTimeout: 10, tag: 'latest'
            }
        }
        stage('Approval for Staging Env Deployment') {
            when {
                branch 'master'
            }
            steps {
               input 'Deploy to Staging Environment?'
               //milestone(1)
            }
        
        }
        stage('Deploy To Staging') {
            when {
                branch 'master'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@darkjenkins.ddns.net \"docker pull darkwunan/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@darkjenkins.ddns.net \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@darkjenkins.ddns.net \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@darkjenkins.ddns.net \"docker run --restart always --name train-schedule -p 8080:8080 -d darkwunan/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
        stage('Application Smoke/Sanity Test') {
            when {
                branch 'master'
            }
            steps {
               input 'Deploy to Production Environment?'
            }
        
        }
         stage('Push Apps Image to Rancher GitOps') {
            when {
                branch 'master'
            }
            steps {
            sh """
            git clone https://github.com/szulkifli-suse/fleet-examples.git --branch=master /tmp/built_${env.BUILD_NUMBER}
            cd /tmp/built_${env.BUILD_NUMBER}/simple/
            sed -i "s/train-schedule:[0-9][0-9]/train-schedule:${env.BUILD_NUMBER}/g" frontend-deployment.yaml
            cat frontend-deployment.yaml
            git add .
            git commit -m "Commit new changes"
            echo "Check what is GIT"
            echo $GIT
            git remote set-url origin https://$GIT@github.com/szulkifli-suse/fleet-examples.git
            git push origin master
            """
            }
        }
    }
}
