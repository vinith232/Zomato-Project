pipeline {
agent {label 'slave'}
    tools {
        jdk 'jdk'
        nodejs 'nodejs'
    }
    environment {
        SCANNER_HOME = tool 'sonar'
        AWS_REGION = 'ap-south-1'
        ECR_REPO = 'zomato'
        ACCOUNT_ID = '884394270671' // replace with your AWS account ID
        IMAGE_TAG = "V${env.BUILD_NUMBER}"
        REPO_URI = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    }

    stages {
        stage("clean") {
            steps {
                cleanWs()
            }
        }
        stage("clone"){
            steps{
                git "https://github.com/vinith232/Zomato-Project.git"
            }
    }
    stage("scan"){
        steps{
            withSonarQubeEnv('sonar'){
                sh'''$SCANNER_HOME/bin/sonar-scanner \
                -Dsonar.projectName=zomato \
                -Dsonar.projectKey=zomato12
                '''
            }
        }
    }
   stage("quality check"){
        steps{
             waitForQualityGate abortPipeline: false, credentialsId: 'sonarkey'
        }
    }
    stage("npm"){
        steps{
            sh'''
            npm install 
            npm test -- --passWithNoTests
            '''
        }
    }
    stage("trivy scan"){
        steps{
            sh "trivy fs .>trivy.txt" 
        }
    }
    stage('Build Docker Image') {
            steps {
                sh 'docker build -t appimage .'
            }
        }

        stage('Scan Docker Image') {
            steps {
                sh 'trivy image appimage'
            }
        }
   
    stage("Docker build image"){
        steps{
            sh "docker tag appimage ${REPO_URI}:${IMAGE_TAG}"  
        }
    }
    stage("Tag and Push to ECR") {
    steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',credentialsId: 'aws_cred']])
        {
            sh """
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin ${REPO_URI}

                docker tag ${REPO_URI}:${IMAGE_TAG} ${REPO_URI}:${IMAGE_TAG}

                docker push ${REPO_URI}:${IMAGE_TAG}
            """
        }
    }
}
stage('Deploy to ECS') {
    steps {
        withAWS(credentials: 'aws_cred', region: 'ap-south-1') {
            script {
                def newImage = "${REPO_URI}:${IMAGE_TAG}"

                sh """
                echo "Deploying image: ${newImage}"

                aws ecs describe-task-definition --task-definition zomato --region ${AWS_REGION} > task-def.json

                cat task-def.json | jq '.taskDefinition |
                    del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy) |
                    .containerDefinitions[0].image = "${newImage}"' > new-task-def.json

                aws ecs register-task-definition --cli-input-json file://new-task-def.json --region ${AWS_REGION}

                newRevision=\$(aws ecs describe-task-definition --task-definition zomato --region ${AWS_REGION} | jq -r '.taskDefinition.revision')

                aws ecs update-service --cluster zomato-cluster --service zomato-service --task-definition zomato:\$newRevision --region ${AWS_REGION}
                """
            }
        }
    }
}



}
    
}
