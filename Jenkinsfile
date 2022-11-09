pipeline {
    parameters {
        string(name: "appVersion", defaultValue: "1.0")
        booleanParam(name: 'autoApprove', defaultValue: false, description: 'Automatically run apply after generating plan?')
        booleanParam(name: 'destroy', defaultValue: false, description: 'Destroy Terraform build?')
    }

    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
		DOCKERHUB_CREDENTIALS=credentials('dockerhub')
        sshCredsID = 'AWS_UBUNTU_INSTANCE_SSH_KEY'
    }

    stages {

        stage('Plan') {
            steps {
                sh 'terraform init -input=false'
                sh "terraform plan -input=false -out tfplan"
            }
        }

        stage('Apply') {
            steps {
                sh "terraform apply"
                script {
                    builderDnsName = sh(
                       script: "terraform output -raw builder_dns_name",
                       returnStdout: true
                    ).trim()
                    webserverDnsName = sh(
                       script: "terraform output -raw webserver_dns_name",
                       returnStdout: true
                    ).trim()
                }
            }
        } 


        stage('Ansible playbook') {

            steps {
                sh "if [ -f hosts ]; then rm hosts; fi"
                sh "echo '[builder]' >> hosts"
                sh "[ '${builderDnsName}' = '' ] || echo ${builderDnsName} >> hosts"
                sh "echo '[webserver]' >> hosts"
                sh "[ '${webserverDnsName}' = '' ] || echo ${webserverDnsName} >> hosts"
                ansiblePlaybook(
                    playbook: 'prepare-instances.yml',
                    inventory: 'hosts',
                    credentialsId: "${sshCredsID}",
                    disableHostKeyChecking: false,
                    become: true,
                )
            }
        } 


        stage('Builder fetch and build') {
            environment {
                DOCKER_HOST="ssh://ubuntu@${builderDnsName}"
            }
            steps {
                sshagent( credentials:["${sshCredsID}"] ) {
                    sh "docker build -t brandani/mywebapp_boxfuser:latest ."
                    sh "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"
                    sh 'docker push brandani/mywebapp_boxfuser:latest'

                }
            }

        stage('Webserver stop and remove') {
            environment {
                DOCKER_HOST="ssh://ubuntu@${webserverDnsName}"
            }
            steps {
                sshagent( credentials:["${sshCredsID}"] ) {
                    sh "for ID in \$(docker ps -q); do docker stop \$ID; done"
                    sh "for ID in \$(docker ps -a -q); do docker rm \$ID; done"
                    sh "for ID in \$(docker images -q); do docker rmi \$ID; done"}
                }
            }

        stage('Run Docker container on remote hosts') {
            environment {
                DOCKER_HOST="ssh://ubuntu@${webserverDnsName}"
            }
            steps {
                sshagent( credentials:["${sshCredsID}"] ) {
                    sh "docker -H run -d -p 8080:8080 brandani/mywebapp_boxfuser"}
                }
            }


         // stage Builder fetch and build

        // stage('Builder push to the registry') {
        //     when {
        //         not {
        //             equals( expected: true, actual: params.destroy )
        //         }
        //     }

        //     environment {
        //         DOCKER_HOST="ssh://ubuntu@${builderDnsName}"
        //     }

        //     steps {
        //         sshagent( credentials:["${sshCredsID}"] ) {
        //             withDockerRegistry( [credentialsId:"${registryCredsID}", url:"https://${registryHost}"] ) {
        //                 sh "docker push ${registryHost}/${repositoryName}:${params.appVersion}"
        //             }
        //         }
        //     }
        // } // stage Builder push to the registry

        ///////////////////////////////
        /// Webserver stages
        ///////////////////////////////



        // stage('Webserver pull and start') {
        //     when {
        //         not {
        //             equals( expected: true, actual: params.destroy )
        //         }
        //     }

        //     environment {
        //         DOCKER_HOST="ssh://ubuntu@${webserverDnsName}"
        //     }

        //     steps {
        //         sshagent( credentials:["${sshCredsID}"] ) {
        //             withDockerRegistry( [credentialsId:"${registryCredsID}", url:"https://${registryHost}"] ) {
        //                 sh "docker pull ${registryHost}/${repositoryName}:${params.appVersion}"
        //             }
        //             sh "docker run -p 80:5000 -d ${registryHost}/${repositoryName}:${params.appVersion}"
        //             echo "########################################################################################"
        //             echo "### curl http://${webserverDnsName}"
        //             echo "########################################################################################"
        //         }
        //     }
        // } // stage Webserver pull and start

        ///////////////////////////////////
        /// Terraform Destroy and cleanws
        ///////////////////////////////////

        // stage('Destroy') {
        //     when {
        //         equals( expected: true, actual: params.destroy )
        //     }
        
        //     steps {
        //         sh 'terraform destroy --auto-approve'
        //         cleanWs()
        //     }
        // } // stage Destroy

    } // stages
}