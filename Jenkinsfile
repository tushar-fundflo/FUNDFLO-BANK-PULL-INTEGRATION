pipeline{
    agent any
    parameters{
        choice(name:'AWS_ENV', choices:['KMR','DEV','QA','UAT'],description:'From which environment do you want to deploy?')
        choice(name:'AWS_REGION', choices:['ap-south-1'],description:'From which region do you want to deploy?')
        choice(name:'EB_APP_NAME', choices:['FUNDFLO-BANK-PULL-INTEGRATION','FUNDFLO-TP-INTEGRATION'],description:'From which elasticbeanstalk application do you want to deploy?')
    }
    stages{
        stage('Resources Check or Create'){
            steps{
                script{
                    
                    if (params.AWS_ENV=='DEV'){
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId:params.AWS_ENV]]){
                            def elasticbeanstalkVpcExists = sh(returnStdout: true, script: "aws ec2 describe-vpcs --filters Name=tag:Name,Values=FUND-${params.AWS_ENV}-VPC --query 'Vpcs[0].VpcId' --output text").trim()
                            def elasticbeanstalkServiceRoleExists = sh(returnStdout: true, script: "aws iam list-roles --query 'Roles[?RoleName==`FUND-${params.AWS_ENV}-EBS-ServiceRole`] | [0].RoleName' --output text").trim()
                            def elasticbeanstalkInstanceProfileExists = sh(returnStdout:true, script: "aws iam list-instance-profiles --query 'InstanceProfiles[?InstanceProfileName==`FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile`] | [0].InstanceProfileName' --output text").trim()
                            def elasticbeanstalkApplicationExists = sh(returnStdout:true, script: "aws elasticbeanstalk describe-applications --application-names ${params.EB_APP_NAME} --query 'Applications[0].ApplicationName' --output text").trim()
                            def elasticbeanstalkEnviornment1Exists = sh (returnStdout:true, script: "aws elasticbeanstalk describe-environments --environment-names ${params.EB_APP_NAME}-env-1 --query 'Environments[0].EnvironmentName' --output text").trim()
                            def elasticbeanstalkEnviornment2Exists = sh (returnStdout:true, script: "aws elasticbeanstalk describe-environments --environment-names ${params.EB_APP_NAME}-env-2 --query 'Environments[0].EnvironmentName' --output text").trim()
                            
                            checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/tushar-fundflo/Resources.git']])
                            if (elasticbeanstalkVpcExists!='None') {
                                echo 'Elasticbeanstalk Vpc Exists'
                            }else{
                            echo'Elasticbeanstalk Vpc Not Exists'
                            dir ('VPC'){
                                echo'Creating VPC'
                                sh 'terraform init'
                                sh "terraform plan -var='vpc_name=FUND-${params.AWS_ENV}'"
                                sh "terraform apply --auto-approve -var='vpc_name=FUND-${params.AWS_ENV}'"
                            }
                            }
                            if (elasticbeanstalkServiceRoleExists!='None'){
                                echo 'Elasticbeanstalk ServiceRole Exists'
                            }else{
                                echo 'Elasticbeanstalk ServiceRole Not Exists'
                                dir('EB-SERVICE_ROLE'){
                                    echo'Creating Elasticbeanstalk ServiceRole'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_service_role_name=FUND-${params.AWS_ENV}-EBS-ServiceRole'"
                                    sh "terraform apply --auto-approve -var='eb_service_role_name=FUND-${params.AWS_ENV}-EBS-ServiceRole'"
                                }
                            }
                            if (elasticbeanstalkInstanceProfileExists!='None') {
                                echo 'Elasticbeanstalk EC2InstanceProfile Exists'
                            }else{
                                echo 'Elasticbeanstalk EC2InstanceProfile Not Exists'
                               dir('EB-EC2_INSTANCE_PROFILE'){
                                    echo'Creating Elasticbeanstalk EC2InstanceProfile'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_ec2_instance_profile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile'"
                                    sh "terraform apply --auto-approve -var='eb_ec2_instance_profile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile'"
                                }

                            }
                            if (elasticbeanstalkApplicationExists!='None') {
                                echo 'Elasticbeanstalk Application Exists'
                            }else{
                                echo 'Elasticbeanstalk Application Not Exists'
                                dir('EB-APPLICATiON'){
                                    echo'Creating Elasticbeanstalk Application'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_application_name=${params.EB_APP_NAME}'"
                                    sh "terraform apply --auto-approve -var='eb_application_name=${params.EB_APP_NAME}'"
                                }
                            }
                            if (elasticbeanstalkEnviornment1Exists!='None'){
                                echo 'Elasticbeanstalk Enviornment 1 Exists'
                            }else {
                                echo 'Elasticbeanstalk Enviornment 1 Not Exists'
                                dir('EB-ENVIRONMENT-1'){
                                    echo 'Creating Elasticbeanstalk Enviornment 1'
                                    def publicsubnet1 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet1a`]].SubnetId' --output text").trim()
                                    def publicsubnet2 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet2b`]].SubnetId' --output text").trim()
                            
                                    sh "terraform init"
                                    sh "terraform plan \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-1' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"
                                    sh "terraform apply --auto-approve \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-1' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"

                                }
                            }
                            if (elasticbeanstalkEnviornment2Exists!='None'){
                                echo 'Elasticbeanstalk Enviornment 2 Exists'
                            }else {
                                echo 'Elasticbeanstalk Enviornment 2 Not Exists'
                                dir('EB-ENVIRONMENT-2'){
                                    echo 'Creating Elasticbeanstalk Enviornment 2'
                                    def publicsubnet1 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet1a`]].SubnetId' --output text").trim()
                                    def publicsubnet2 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet2b`]].SubnetId' --output text").trim()
                            
                                    sh "terraform init"
                                    sh "terraform plan \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-2' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"
                                    sh "terraform apply --auto-approve \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-2' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"

                                }
                                
                            }
                        }
                    } else if (params.AWS_ENV=='QA'){
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId:params.AWS_ENV]]){
                            def elasticbeanstalkVpcExists = sh(returnStdout: true, script: "aws ec2 describe-vpcs --filters Name=tag:Name,Values=FUND-${params.AWS_ENV}-VPC --query 'Vpcs[0].VpcId' --output text").trim()
                            def elasticbeanstalkServiceRoleExists = sh(returnStdout: true, script: "aws iam list-roles --query 'Roles[?RoleName==`FUND-${params.AWS_ENV}-EBS-ServiceRole`] | [0].RoleName' --output text").trim()
                            def elasticbeanstalkInstanceProfileExists = sh(returnStdout:true, script: "aws iam list-instance-profiles --query 'InstanceProfiles[?InstanceProfileName==`FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile`] | [0].InstanceProfileName' --output text").trim()
                            def elasticbeanstalkApplicationExists = sh(returnStdout:true, script: "aws elasticbeanstalk describe-applications --application-names ${params.EB_APP_NAME} --query 'Applications[0].ApplicationName' --output text").trim()
                            def elasticbeanstalkEnviornment1Exists = sh (returnStdout:true, script: "aws elasticbeanstalk describe-environments --environment-names ${params.EB_APP_NAME}-env-1 --query 'Environments[0].EnvironmentName' --output text").trim()
                            def elasticbeanstalkEnviornment2Exists = sh (returnStdout:true, script: "aws elasticbeanstalk describe-environments --environment-names ${params.EB_APP_NAME}-env-2 --query 'Environments[0].EnvironmentName' --output text").trim()
                            
                            checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/tushar-fundflo/Resources.git']])
                            if (elasticbeanstalkVpcExists!='None') {
                                echo 'Elasticbeanstalk Vpc Exists'
                            }else{
                            echo'Elasticbeanstalk Vpc Not Exists'
                            dir ('VPC'){
                                echo'Creating VPC'
                                sh 'terraform init'
                                sh "terraform plan -var='vpc_name=FUND-${params.AWS_ENV}'"
                                sh "terraform apply --auto-approve -var='vpc_name=FUND-${params.AWS_ENV}'"
                            }
                            }
                            if (elasticbeanstalkServiceRoleExists!='None'){
                                echo 'Elasticbeanstalk ServiceRole Exists'
                            }else{
                                echo 'Elasticbeanstalk ServiceRole Not Exists'
                                dir('EB-SERVICE_ROLE'){
                                    echo'Creating Elasticbeanstalk ServiceRole'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_service_role_name=FUND-${params.AWS_ENV}-EBS-ServiceRole'"
                                    sh "terraform apply --auto-approve -var='eb_service_role_name=FUND-${params.AWS_ENV}-EBS-ServiceRole'"
                                }
                            }
                            if (elasticbeanstalkInstanceProfileExists!='None') {
                                echo 'Elasticbeanstalk EC2InstanceProfile Exists'
                            }else{
                                echo 'Elasticbeanstalk EC2InstanceProfile Not Exists'
                               dir('EB-EC2_INSTANCE_PROFILE'){
                                    echo'Creating Elasticbeanstalk EC2InstanceProfile'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_ec2_instance_profile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile'"
                                    sh "terraform apply --auto-approve -var='eb_ec2_instance_profile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile'"
                                }

                            }
                            if (elasticbeanstalkApplicationExists!='None') {
                                echo 'Elasticbeanstalk Application Exists'
                            }else{
                                echo 'Elasticbeanstalk Application Not Exists'
                                dir('EB-APPLICATiON'){
                                    echo'Creating Elasticbeanstalk Application'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_application_name=${params.EB_APP_NAME}'"
                                    sh "terraform apply --auto-approve -var='eb_application_name=${params.EB_APP_NAME}'"
                                }
                            }
                            if (elasticbeanstalkEnviornment1Exists!='None'){
                                echo 'Elasticbeanstalk Enviornment 1 Exists'
                            }else {
                                echo 'Elasticbeanstalk Enviornment 1 Not Exists'
                                dir('EB-ENVIRONMENT-1'){
                                    echo 'Creating Elasticbeanstalk Enviornment 1'
                                    def publicsubnet1 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet1a`]].SubnetId' --output text").trim()
                                    def publicsubnet2 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet2b`]].SubnetId' --output text").trim()
                            
                                    sh "terraform init"
                                    sh "terraform plan \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-1' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"
                                    sh "terraform apply --auto-approve \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-1' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"

                                }
                            }
                            if (elasticbeanstalkEnviornment2Exists!='None'){
                                echo 'Elasticbeanstalk Enviornment 2 Exists'
                            }else {
                                echo 'Elasticbeanstalk Enviornment 2 Not Exists'
                                dir('EB-ENVIRONMENT-2'){
                                    echo 'Creating Elasticbeanstalk Enviornment 2'
                                    def publicsubnet1 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet1a`]].SubnetId' --output text").trim()
                                    def publicsubnet2 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet2b`]].SubnetId' --output text").trim()
                            
                                    sh "terraform init"
                                    sh "terraform plan \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-2' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"
                                    sh "terraform apply --auto-approve \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-2' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"

                                }
                                
                            }
                        }
                    }else if (params.AWS_ENV=='UAT'){
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId:params.AWS_ENV]]){
                            def elasticbeanstalkVpcExists = sh(returnStdout: true, script: "aws ec2 describe-vpcs --filters Name=tag:Name,Values=FUND-${params.AWS_ENV}-VPC --query 'Vpcs[0].VpcId' --output text").trim()
                            def elasticbeanstalkServiceRoleExists = sh(returnStdout: true, script: "aws iam list-roles --query 'Roles[?RoleName==`FUND-${params.AWS_ENV}-EBS-ServiceRole`] | [0].RoleName' --output text").trim()
                            def elasticbeanstalkInstanceProfileExists = sh(returnStdout:true, script: "aws iam list-instance-profiles --query 'InstanceProfiles[?InstanceProfileName==`FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile`] | [0].InstanceProfileName' --output text").trim()
                            def elasticbeanstalkApplicationExists = sh(returnStdout:true, script: "aws elasticbeanstalk describe-applications --application-names ${params.EB_APP_NAME} --query 'Applications[0].ApplicationName' --output text").trim()
                            def elasticbeanstalkEnviornment1Exists = sh (returnStdout:true, script: "aws elasticbeanstalk describe-environments --environment-names ${params.EB_APP_NAME}-env-1 --query 'Environments[0].EnvironmentName' --output text").trim()
                            def elasticbeanstalkEnviornment2Exists = sh (returnStdout:true, script: "aws elasticbeanstalk describe-environments --environment-names ${params.EB_APP_NAME}-env-2 --query 'Environments[0].EnvironmentName' --output text").trim()
                            
                            checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/tushar-fundflo/Resources.git']])
                            if (elasticbeanstalkVpcExists!='None') {
                                echo 'Elasticbeanstalk Vpc Exists'
                            }else{
                            echo'Elasticbeanstalk Vpc Not Exists'
                            dir ('VPC'){
                                echo'Creating VPC'
                                sh 'terraform init'
                                sh "terraform plan -var='vpc_name=FUND-${params.AWS_ENV}'"
                                sh "terraform apply --auto-approve -var='vpc_name=FUND-${params.AWS_ENV}'"
                            }
                            }
                            if (elasticbeanstalkServiceRoleExists!='None'){
                                echo 'Elasticbeanstalk ServiceRole Exists'
                            }else{
                                echo 'Elasticbeanstalk ServiceRole Not Exists'
                                dir('EB-SERVICE_ROLE'){
                                    echo'Creating Elasticbeanstalk ServiceRole'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_service_role_name=FUND-${params.AWS_ENV}-EBS-ServiceRole'"
                                    sh "terraform apply --auto-approve -var='eb_service_role_name=FUND-${params.AWS_ENV}-EBS-ServiceRole'"
                                }
                            }
                            if (elasticbeanstalkInstanceProfileExists!='None') {
                                echo 'Elasticbeanstalk EC2InstanceProfile Exists'
                            }else{
                                echo 'Elasticbeanstalk EC2InstanceProfile Not Exists'
                               dir('EB-EC2_INSTANCE_PROFILE'){
                                    echo'Creating Elasticbeanstalk EC2InstanceProfile'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_ec2_instance_profile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile'"
                                    sh "terraform apply --auto-approve -var='eb_ec2_instance_profile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile'"
                                }

                            }
                            if (elasticbeanstalkApplicationExists!='None') {
                                echo 'Elasticbeanstalk Application Exists'
                            }else{
                                echo 'Elasticbeanstalk Application Not Exists'
                                dir('EB-APPLICATiON'){
                                    echo'Creating Elasticbeanstalk Application'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_application_name=${params.EB_APP_NAME}'"
                                    sh "terraform apply --auto-approve -var='eb_application_name=${params.EB_APP_NAME}'"
                                }
                            }
                            if (elasticbeanstalkEnviornment1Exists!='None'){
                                echo 'Elasticbeanstalk Enviornment 1 Exists'
                            }else {
                                echo 'Elasticbeanstalk Enviornment 1 Not Exists'
                                dir('EB-ENVIRONMENT-1'){
                                    echo 'Creating Elasticbeanstalk Enviornment 1'
                                    def publicsubnet1 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet1a`]].SubnetId' --output text").trim()
                                    def publicsubnet2 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet2b`]].SubnetId' --output text").trim()
                            
                                    sh "terraform init"
                                    sh "terraform plan \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-1' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"
                                    sh "terraform apply --auto-approve \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-1' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"

                                }
                            }
                            if (elasticbeanstalkEnviornment2Exists!='None'){
                                echo 'Elasticbeanstalk Enviornment 2 Exists'
                            }else {
                                echo 'Elasticbeanstalk Enviornment 2 Not Exists'
                                dir('EB-ENVIRONMENT-2'){
                                    echo 'Creating Elasticbeanstalk Enviornment 2'
                                    def publicsubnet1 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet1a`]].SubnetId' --output text").trim()
                                    def publicsubnet2 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet2b`]].SubnetId' --output text").trim()
                            
                                    sh "terraform init"
                                    sh "terraform plan \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-2' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"
                                    sh "terraform apply --auto-approve \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-2' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"

                                }
                                
                            }
                        }
                    }else if (params.AWS_ENV=='KMR'){
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId:params.AWS_ENV]]){
                            def elasticbeanstalkVpcExists = sh(returnStdout: true, script: "aws ec2 describe-vpcs --filters Name=tag:Name,Values=FUND-${params.AWS_ENV}-VPC --query 'Vpcs[0].VpcId' --output text").trim()
                            def elasticbeanstalkServiceRoleExists = sh(returnStdout: true, script: "aws iam list-roles --query 'Roles[?RoleName==`FUND-${params.AWS_ENV}-EBS-ServiceRole`] | [0].RoleName' --output text").trim()
                            def elasticbeanstalkInstanceProfileExists = sh(returnStdout:true, script: "aws iam list-instance-profiles --query 'InstanceProfiles[?InstanceProfileName==`FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile`] | [0].InstanceProfileName' --output text").trim()
                            def elasticbeanstalkApplicationExists = sh(returnStdout:true, script: "aws elasticbeanstalk describe-applications --application-names ${params.EB_APP_NAME} --query 'Applications[0].ApplicationName' --output text").trim()
                            def elasticbeanstalkEnviornment1Exists = sh (returnStdout:true, script: "aws elasticbeanstalk describe-environments --environment-names ${params.EB_APP_NAME}-env-1 --query 'Environments[0].EnvironmentName' --output text").trim()
                            def elasticbeanstalkEnviornment2Exists = sh (returnStdout:true, script: "aws elasticbeanstalk describe-environments --environment-names ${params.EB_APP_NAME}-env-2 --query 'Environments[0].EnvironmentName' --output text").trim()
                            
                            checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/tushar-fundflo/Resources.git']])
                            if (elasticbeanstalkVpcExists!='None') {
                                echo 'Elasticbeanstalk Vpc Exists'
                            }else{
                            echo'Elasticbeanstalk Vpc Not Exists'
                            dir ('VPC'){
                                echo'Creating VPC'
                                sh 'terraform init'
                                sh "terraform plan -var='vpc_name=FUND-${params.AWS_ENV}'"
                                sh "terraform apply --auto-approve -var='vpc_name=FUND-${params.AWS_ENV}'"
                            }
                            }
                            if (elasticbeanstalkServiceRoleExists!='None'){
                                echo 'Elasticbeanstalk ServiceRole Exists'
                            }else{
                                echo 'Elasticbeanstalk ServiceRole Not Exists'
                                dir('EB-SERVICE_ROLE'){
                                    echo'Creating Elasticbeanstalk ServiceRole'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_service_role_name=FUND-${params.AWS_ENV}-EBS-ServiceRole'"
                                    sh "terraform apply --auto-approve -var='eb_service_role_name=FUND-${params.AWS_ENV}-EBS-ServiceRole'"
                                }
                            }
                            if (elasticbeanstalkInstanceProfileExists!='None') {
                                echo 'Elasticbeanstalk EC2InstanceProfile Exists'
                            }else{
                                echo 'Elasticbeanstalk EC2InstanceProfile Not Exists'
                               dir('EB-EC2_INSTANCE_PROFILE'){
                                    echo'Creating Elasticbeanstalk EC2InstanceProfile'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_ec2_instance_profile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile'"
                                    sh "terraform apply --auto-approve -var='eb_ec2_instance_profile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile'"
                                }

                            }
                            if (elasticbeanstalkApplicationExists!='None') {
                                echo 'Elasticbeanstalk Application Exists'
                            }else{
                                echo 'Elasticbeanstalk Application Not Exists'
                                dir('EB-APPLICATiON'){
                                    echo'Creating Elasticbeanstalk Application'
                                    sh 'terraform init'
                                    sh "terraform plan -var='eb_application_name=${params.EB_APP_NAME}'"
                                    sh "terraform apply --auto-approve -var='eb_application_name=${params.EB_APP_NAME}'"
                                }
                            }
                            if (elasticbeanstalkEnviornment1Exists!='None'){
                                echo 'Elasticbeanstalk Enviornment 1 Exists'
                            }else {
                                echo 'Elasticbeanstalk Enviornment 1 Not Exists'
                                dir('EB-ENVIRONMENT-1'){
                                    echo 'Creating Elasticbeanstalk Enviornment 1'
                                    def publicsubnet1 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet1a`]].SubnetId' --output text").trim()
                                    def publicsubnet2 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet2b`]].SubnetId' --output text").trim()
                            
                                    sh "terraform init"
                                    sh "terraform plan \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-1' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"
                                    sh "terraform apply --auto-approve \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-1' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"

                                }
                            }
                            if (elasticbeanstalkEnviornment2Exists!='None'){
                                echo 'Elasticbeanstalk Enviornment 2 Exists'
                            }else {
                                echo 'Elasticbeanstalk Enviornment 2 Not Exists'
                                dir('EB-ENVIRONMENT-2'){
                                    echo 'Creating Elasticbeanstalk Enviornment 2'
                                    def publicsubnet1 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet1a`]].SubnetId' --output text").trim()
                                    def publicsubnet2 = sh(returnStdout:true, script: "aws ec2 describe-subnets --filters Name=vpc-id,Values=${elasticbeanstalkVpcExists} --query 'Subnets[?Tags[?Key==`Name` && Value==`FUND-${params.AWS_ENV}-PublicSubnet2b`]].SubnetId' --output text").trim()
                            
                                    sh "terraform init"
                                    sh "terraform plan \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-2' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"
                                    sh "terraform apply --auto-approve \
                                        -var='eb_application_name=${params.EB_APP_NAME}' \
                                        -var='eb_environment_name=${params.EB_APP_NAME}-env-2' \
                                        -var='eb_environment_platform=arn:aws:elasticbeanstalk:${params.AWS_REGION}::platform/Node.js 18 running on 64bit Amazon Linux 2023/6.0.3' \
                                        -var='eb_environment_vpc_id=${elasticbeanstalkVpcExists}' \
                                        -var='eb_environment_instanceprofile_name=FUND-${params.AWS_ENV}-EBS-EC2InstanceProfile' \
                                        -var='eb_environment_subnets_id=${publicsubnet1},${publicsubnet2}' \
                                        -var='eb_environment_instance_type=t3.micro' \
                                        -var='eb_environment_autoscaling_maxcount=2'"

                                }
                                
                            }
                        }
                    }
                }    
            }
        }
        stage ('Deploy To AWS'){
            steps{
                script{
                    
                    def directoryExists = fileExists("${params.EB_APP_NAME}")
                    if (directoryExists){
                        echo "Directory Exists ${params.EB_APP_NAME}"
                    }else {
                        sh "mkdir ${params.EB_APP_NAME}"
                    }
                    
                    def appDirectory = params.EB_APP_NAME.toString()
                    def scmVars = checkout([$class: 'GitSCM', branches: [[name: '*/main']], 
                                            userRemoteConfigs: [[url: 'https://github.com/tushar-fundflo/FUNDFLO-BANK-PULL-INTEGRATION.git']]])
                    scmVars = scmVars[0]
                    scmVars.dir(appDirectory)

                    // def appDirectory = params.EB_APP_NAME.toString()
                    // checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/tushar-fundflo/FUNDFLO-BANK-PULL-INTEGRATION.git']]).dir(appDirectory)

                    sh "zip -r version-${BUILD_NUMBER}.zip ${params.EB_APP_NAME}"

                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId:params.AWS_ENV, region:params.AWS_REGION]]) {
                        def accountNumber = sh(returnStdout: true, script: 'aws sts get-caller-identity --query "Account" --output text').trim()
                        sh "aws s3 cp version-${BUILD_NUMBER}.zip s3://elasticbeanstalk-${params.AWS_REGION}-${bucketname} --region ${params.AWS_REGION}"
                        sh """aws elasticbeanstalk create-application-version --application-name "${params.EB_APP_NAME}" \
                        --version-label "version-${BUILD_NUMBER}" \
                        --description "Build created from JENKINS. Job:${JOB_NAME}, BuildId:${BUILD_DISPLAY_NAME}" \
                        --source-bundle S3Bucket=${bucketname},S3Key=version-${BUILD_NUMBER}.zip \
                        --region ${params.AWS_REGION}"""
                        sh "aws elasticbeanstalk update-environment --environment-name \"${params.EB_APP_NAME}\" --version-label \"version-${BUILD_NUMBER}\" --region ${params.AWS_REGION}"
                    } 
                }
            }

        }
        stage ('Clean Up'){
            steps {
                sh "rm /var/lib/jenkins/workspace/Tushar_main/version-${BUILD_NUMBER}.zip"
            }
        }
        
    }
}
