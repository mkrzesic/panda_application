pipeline {
    agent {
        label 'Slave'
    }

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "m3"
        terraform 'Terraform'
    }
    
    environment {
        IMAGE = readMavenPom().getArtifactId()
        VERSION = readMavenPom().getVersion()
        ANSIBLE = tool name: 'Ansible', type: 'com.cloudbees.jenkins.plugins.customtools.CustomTool'
    }
    
    stages {
        
        stage('Clear running app') {
            steps {
                // Clear previous instances of app built
                sh "docker rm -f pandaapp || true"
            }
        }
        
        stage('Get code') {
            steps {
                // Get Panda app from GitHub
                checkout scm
            }
        }
        
        stage('Build and Junit') {
            steps {
                // Build app on slave.
                sh "mvn clean install"
            }
        }
        
        stage('Build Docker image') {
            steps {
                sh "mvn package -Pdocker -Dmaven.test.skip=true"
            }
        }
        
        stage('Run Docker app') {
            steps {
                sh "docker run -d -p 0.0.0.0:8080:8080 --name pandaapp -t ${IMAGE}:${VERSION}"
            }
        }
        
        stage('Test Selenium') {
            steps {
                // Run Selenium tests
                sh "mvn test -Pselenium"
            }
        }
        
        stage('Deploy jar to Artifactory') {
            steps {
                // Deploy
                configFileProvider([configFile(fileId: '41728bbe-f0ca-44dd-b5b4-cda2651bb58c', variable: 'mavensettings')]) {

                    sh "mvn -s $mavensettings deploy -Dmaven.test.skip=true -e"
                    
                }
            }
        }

        stage('Run terraform') {
            steps {
                dir('infrastructure/terraform') { 
                    sh 'terraform init && terraform apply -var-file panda.tfvars -auto-approve'
                } 
            }
        }

        stage('Copy Ansible role') {
            steps {
                sh 'cp -r infrastructure/ansible/panda/ /etc/ansible/roles/'
            }
        }

        stage('Run Ansible') {
            steps {
                dir('infrastructure/ansible') { 
                sh 'chmod 600 ../core5.pem'
                sh 'ansible-playbook -i ./inventory playbook.yml'
                } 
            }
        }
    }
        
    post {
        always {
            sh 'docker stop pandaapp'
            deleteDir()
        }
    }
        
}
