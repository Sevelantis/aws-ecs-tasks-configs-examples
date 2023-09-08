Jpipeline {
    agent {
        label 'agent_007'
    }
    
    stages {
        stage("Setup Environment") {
            steps {
                script {
                    CLUSTER = 'global-cluster'
                    REGION = 'eu-west-3'
                    switch(env.BRANCH_NAME) { 
                        case 'master': 
                            IMAGE = 'human-detection-module-prod'
                            SERVICE = 'human-detection-module-prod'
                            TASK = 'human-detection-module-prod'
                            break
                        case "dev": 
                            IMAGE = 'human-detection-module-dev'
                            SERVICE = 'human-detection-module-dev'
                            TASK = 'human-detection-module-dev'
                            break
                        default:
                            println("Branch value error: " + env.BRANCH_NAME)
                            currentBuild.getRawBuild().getExecutor().interrupt(Result.FAILURE)
                    }
                }
            
            }
        }
      
        stage("SonarQube App Code Analysis") {
            steps {
                withSonarQubeEnv('sonar_master') {
                    withCredentials([string(credentialsId: 'sonar_surveillance4se_token', variable: 'sonar_token')]) {
                        sh 'echo "analyze python codee"'
                    }
                }
            }
        }
        
        stage("Start test: broker & redis") {
            steps {
                sh '/usr/local/bin/docker-compose up -d'
            }
        } 
        
        stage("App Tests") {
            steps {
                sh '/usr/local/bin/pytest'
                sh '/usr/local/bin/docker-compose down'
            }
        }
        
        stage("Build Docker Image") {
            steps{
                script{
                   image=docker.build(IMAGE)
                }
            }
        }
        
        stage("Upload image to ECR") {
            steps {
                script{
                        docker.withRegistry('https://358682810880.dkr.ecr.eu-west-3.amazonaws.com/${env.IMAGE}', 'ecr:eu-west-3:aws-credentials') {
                            image.push("latest")
                    }
                }
            }
        }
        
        stage('Clone ECS Config files') {
            steps {
                git branch: 'main',
                credentialsId: 'git_repo_ecs_tasks_json_configurations_key',
                url: 'git@github.com:Surveillance4SE/ECS-tasks-json-configurations.git'
            }
        }
        stage("Update ECS") {
            steps{
                withAWS(credentials:'aws-credentials', region:'eu-west-3') {
                    sh '''\
                    for taskarn in $(aws ecs list-tasks --cluster '''+CLUSTER+''' --service '''+SERVICE+''' --desired-status RUNNING --output text --query 'taskArns'); do aws ecs stop-task --cluster '''+CLUSTER+''' --task $taskarn; done;                    
                    aws ecs register-task-definition --family '''+TASK+''' --cli-input-json file://'''+TASK+'''.json
                    TASK_REVISION=`aws ecs describe-task-definition --task-definition '''+TASK+''' | egrep "revision" | tr "/" " " | awk '{print $2}' | sed 's/"$//' | sed 's/[^0-9]*//g'` 
                    aws ecs update-service --cluster '''+CLUSTER+''' --service '''+SERVICE+''' --task-definition '''+TASK+''':${TASK_REVISION} --desired-count 1 --force-new-deployment --network-configuration file://network/'''+TASK+'''.json
                    '''
                }   
            }
        }
    }
    post {
        failure {
            sh '/usr/local/bin/docker-compose down'
        }
    }
}
