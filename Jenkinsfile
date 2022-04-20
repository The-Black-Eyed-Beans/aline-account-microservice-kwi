pipeline {
    agent {
        node {
            label "worker-one"
        }
    }

    tools {
        maven "Maven"
    }

    environment {
        AWS_ID = credentials("AWS-ACCOUNT-ID")
        REGION = credentials("REGION-KWI")
        APP = "account"
        PROJECT = "account-microservice"
        COMMIT_HASH = "${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}"
        APP_PORT = 8072
        JFROG_URL = "44.204.62.155:8082"
        JFROG_USER = credentials("ARTIFACTORY-USER-KWI")
        JFROG_PASS = credentials("ARTIFACTORY-PASSWORD-KWI")
        DEPLOYMENT = "EKS"
    }

    stages {
        stage("Test") {
            steps {
                sh "git submodule init"
                sh "git submodule update"
                sh "mvn clean test"
            }
        }

        stage("SonarQube Code Analysis") {
            steps {
                echo "Running SonarQube Analysis..."
                withSonarQubeEnv(installationName: 'beb-sonar-server'){
                    sh "mvn verify sonar:sonar -Dsonar.projectName=${PROJECT}-kwi"
                }
            }
        }

        stage("Quality Gate"){
            steps {
                echo "Waiting for Quality Gate..."
                timeout(time: 5, unit: "MINUTES") {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Package")  {
            steps {
                sh "mvn package -DskipTests"
            }
        }

        stage("Docker Build") {
            steps {
                echo "Authenticating with AWS Credentials..."
                sh "docker context use default"
                sh "aws ecr get-login-password --region ${REGION} --profile keshaun | docker login --username AWS --password-stdin ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com"

                echo "Building Docker Image with Commit Hash as the tag..."
                sh "docker build -t ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH} ."
                sh "docker push ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"

                echo "Building Docker Image with latest tag..."
                sh "docker tag ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH} ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:latest"
                sh "docker push ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:latest"
            }
        }

        stage("Push to Artifactory") {
            steps {
                echo "Logging in to Artifactory..."
                sh "docker login -u ${JFROG_USER} -p ${JFROG_PASS} ${JFROG_URL}"

                echo "Pushing Docker Images to Artifactory..."
                sh "docker tag ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH} ${JFROG_URL}/docker/${PROJECT}:${COMMIT_HASH}"
                sh "docker tag ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:latest ${JFROG_URL}/docker/${PROJECT}:latest"
                sh "docker push -a ${JFROG_URL}/docker/${PROJECT}"

                echo "Pushing JAR files to Artifactory..."
                sh "mvn deploy -DskipTests"
            }
        }

        stage("ECS Deployment") {
            when {
                environment(name: "DEPLOYMENT", value: "ECS")
            }
            steps {
                echo "Deploying ${PROJECT}-kwi..."
                sh '''
                aws cloudformation deploy \
                --stack-name ${PROJECT}-kwi-stack \
                --template-file deploy.json \
                --profile keshaun \
                --capabilities CAPABILITY_IAM \
                --no-fail-on-empty-changeset \
                --parameter-overrides \
                    MicroserviceName=${PROJECT} \
                    AppPort=${APP_PORT} \
                    ImageTag=${COMMIT_HASH}
                '''
            }
        }

        stage("EKS Deployment") {
            when {
                environment(name: "DEPLOYMENT", value: "EKS")
            }
            steps {
                echo "Generating .env..."
                sh """aws secretsmanager get-secret-value --secret-id aline-kwi/dev/secrets/resources --region us-east-1 --profile keshaun | jq -r '.["SecretString"]' | jq '.' | jq -r 'keys[] as \$k | "export \\(\$k)=\\(.[\$k])"' > .env"""
                sh """aws secretsmanager get-secret-value --secret-id aline-kwi/dev/secrets/user-credentials --region us-east-1 --profile keshaun | jq -r '.["SecretString"]' | jq '.' | jq -r 'keys[] as \$k | "export \\(\$k)=\\(.[\$k])"' >> .env"""
                sh """aws secretsmanager get-secret-value --secret-id aline-kwi/dev/secrets/db --region us-east-1 --profile keshaun | jq -r '.["SecretString"]' | jq '.' | jq -r 'keys[] as \$k | "export Db\\(\$k)=\\(.[\$k])"' >> .env"""
                
                sh "echo 'export ImageTag=${COMMIT_HASH}' >> .env"
                sh "echo 'export AppPort=${APP_PORT}' >> .env"
                sh "echo 'export AppName=${APP}' >> .env"
                sh "echo 'export Project=${PROJECT}' >> .env"
                sh "echo 'export AwsId=${AWS_ID}' >> .env"
                sh "echo 'export AwsRegion=${REGION}' >> .env"

                echo "Deploying ${PROJECT}-kwi..."
                sh "aws eks update-kubeconfig --name=aline-kwi-eks --region=us-east-1 --profile keshaun"
                sh ". ./.env && envsubst < deployment.yml | kubectl apply -f -"

                echo "Deleting .env..."
                sh "rm .env"
            }
        }
    }

    post {
        always {
            sh "docker image rm ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
            sh "docker image rm ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:latest"
            sh "docker image rm ${JFROG_URL}/docker/${PROJECT}:${COMMIT_HASH}"
            sh "docker image rm ${JFROG_URL}/docker/${PROJECT}:latest"
            sh "mvn clean"
        }
    }
}