  pipeline {
    agent any

    parameters {
        string(name: 'environment', defaultValue: 'terraform', description: 'Workspace/environment file to use for deployment')
        booleanParam(name: 'autoApprove', defaultValue: true, description: 'Automatically run apply after generating plan?')
        booleanParam(name: 'destroy', defaultValue: false, description: 'Destroy Terraform build?')
        }


    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        }

    stages {
        stage('Get files from Git') {
            steps {
                 script{
                        dir("terraform")
                        {
                            git "https://github.com/EvgenyKiselgof/home.git"
                        }
                    }
                }
            }
                    
        stage('Plan') {
            when {
                not {
                    equals expected: true, actual: params.destroy
                }
            }
            
            steps {
                sh 'terraform init -input=false'
                sh "terraform plan -input=false -out tfplan "
                sh 'terraform show -no-color tfplan > tfplan.txt'
            }
        }
        stage('Approval') {
           when {
               not {
                   equals expected: true, actual: params.autoApprove
               }
               not {
                    equals expected: true, actual: params.destroy
                }
           }
           
                
           steps {
               script {
                    def plan = readFile 'tfplan.txt'
                    input message: "Do you want to apply the plan?",
                    parameters: [text(name: 'Plan', description: 'Please review the plan', defaultValue: plan)]
               }
           }
       }

        stage('Apply') {
            when {
                not {
                    equals expected: true, actual: params.destroy
                }
            }
            
            steps {
                sh "terraform apply -input=false tfplan"
            }
        }
        
        stage('Get kubeconfig') {  
            when {
                not {
                    equals expected: true, actual: params.destroy
                }
            }
            steps {
                sh '''aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)'''
            }
        }
        
        stage('Deploy NGINX') {  
            when {
                not {
                    equals expected: true, actual: params.destroy
                }
            }
            steps {
                sh "kubectl apply -f ./Nginx/index-html-configmap.yaml"
                sh "kubectl apply -f ./Nginx/nginx.yaml"
                sh "kubectl expose deployment nginx-deployment --type=LoadBalancer --name=nginx-service"
            }
        }

        stage('Wait for ELB come up') {
            when {
                not {
                    equals expected: true, actual: params.destroy
                }
            }
            steps {
                echo 'Waiting 5 minutes for deployment to complete prior starting smoke testing'
                sleep 90
            }
        }

        stage('Validate response') {
            when {
                not {
                    equals expected: true, actual: params.destroy
                }
            }
            steps {
                sh '''curl $(kubectl get services nginx-service | tail +2 |awk '{print$4}')'''
            }
        }
        stage('Delete NGINX') {
            when {
                equals expected: true, actual: params.destroy
            }
        steps {
           sh "kubectl delete service nginx-service"
           sh "kubectl delete deployment nginx-deployment"
           sh "kubectl delete configmap index-html-configmap"
        }
        }

        stage('Wait for Removal') {
            when {
                equals expected: true, actual: params.destroy
            }
        
        steps {
                echo 'Waiting 5 minutes for deployment to complete prior starting smoke testing'
                sleep 90
            }
        }   

        stage('Destroy') {
            when {
                equals expected: true, actual: params.destroy
            }
        
        steps {
           sh "terraform destroy --auto-approve"
        }
    }

  }
}