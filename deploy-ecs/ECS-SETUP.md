# Deploying a container-based app to ECS via CI

This walks through how to manually set up the AWS ECS infrastructure for this application, and to deploy the app via CI (including IAM account setup).

It goes through setting up an ECS cluster, deploying a container-based app to it via a task definition and service, and automating the process with a CI tool (TravisCI in this case).

## Process Overview

This process includes:

    - creating an ECS container registry(s) (ECR) for an app, 
    - creating an ECS cluster (or using an existing one), 
    - preparing an ECS task definition file that will deploy and run our app in ECS, by pulling the image from ECR, then creating a service to actually run the app (it will create tasks for that), and make sure it stays running. 

Then, we'll set up a cloud-based CI tool to deploy the app based on check-ins to branches in a VCS. A particular branch (i.e. master, staging, production) will be pushed to, and the CI tool will rebuild the app image, run tests, and push it into the container regiesty (ECR). Then the CI tool will create a revision of the ECS task deinition, and trigger the ECS service to update and redeploy the app based on the task definition. 

### On each deploy

Once everything is set up, here is what happens on each check-in to the master, staging or production branches. They each deploy to demo, staging or production environments respectively: 

- A deployment triggered by push to a VCS branch. CI builds & runs tests. If it is the Demo environemnt, it pushes a new image version to the container registry. If if it staging/production, it promotes the prior built image.
- ecs-deploy.sh creates a new revision of our task definition, for the new version of our app. Each revision only differs by the image tag we specify. We update the task family with the new task revision.
- ecs-deploy updates the running service to use the new task definition revision. Based on how the service was configured, it will smartly stop & start task to update everything to the newest revision. Once the script sees the new task definition running, it’s considered a success and ends.

### Outline:

1. Setting up IAM accounts for ECS management and deployment
1. Setting up ECS for application deployment
1. Setting up CI for automatic build and deployment

## *Part 1.* Setting up IAM accounts for ECS management and deployment

IAM accounts are needed to *create/setup ECS*, and to *deploy the application*.

Steps: 

1. **Create an IAM group user with permission to manage ECS** - create a user and put in a new group called `ecsAdmins`, with these policies:
    - AmazonEC2ContainerServiceFullAccess
    - AWSCertificateManagerFullAccess
    - AmazonRoute53FullAccess 
    - CloudWatchFullAccess
    - New policy called `EC2ContainerServiceAdministration` with these permissions - all operations around ECS that are needed to set it up for an app (i.e. creating the initial service, load balancers):

        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                  {
                  "Effect": "Allow",
                  "Action": [
                      "autoscaling:CreateAutoScalingGroup",
                      "autoscaling:CreateLaunchConfiguration",
                      "autoscaling:CreateOrUpdateTags",
                      "autoscaling:DeleteAutoScalingGroup",
                      "autoscaling:DeleteLaunchConfiguration",
                      "autoscaling:DescribeAutoScalingGroups",
                      "autoscaling:DescribeAutoScalingInstances",
                      "autoscaling:DescribeAutoScalingNotificationTypes",
                      "autoscaling:DescribeLaunchConfigurations",
                      "autoscaling:DescribeScalingActivities",
                      "autoscaling:DescribeTags",
                      "autoscaling:DescribeTriggers",
                      "autoscaling:UpdateAutoScalingGroup",
                      "cloudformation:CreateStack",
                      "cloudformation:DescribeStack*",
                      "cloudformation:DeleteStack",
                      "cloudformation:UpdateStack",
                      "cloudwatch:GetMetricStatistics",
                      "cloudwatch:ListMetrics",
                      "ec2:AssociateRouteTable",
                      "ec2:AttachInternetGateway",
                      "ec2:AuthorizeSecurityGroupIngress",
                      "ec2:CreateInternetGateway",
                      "ec2:CreateKeyPair",
                      "ec2:CreateNetworkInterface",
                      "ec2:CreateRoute",
                      "ec2:CreateRouteTable",
                      "ec2:CreateSecurityGroup",
                      "ec2:CreateSubnet",
                      "ec2:CreateTags",
                      "ec2:CreateVpc",
                      "ec2:DeleteInternetGateway",
                      "ec2:DeleteRoute",
                      "ec2:DeleteRouteTable",
                      "ec2:DeleteSecurityGroup",
                      "ec2:DeleteSubnet",
                      "ec2:DeleteTags",
                      "ec2:DeleteVpc",
                      "ec2:DescribeAccountAttributes",
                      "ec2:DescribeAvailabilityZones",
                      "ec2:DescribeInstances",
                      "ec2:DescribeInternetGateways",
                      "ec2:DescribeKeyPairs",
                      "ec2:DescribeNetworkInterface",
                      "ec2:DescribeRouteTables",
                      "ec2:DescribeSecurityGroups",
                      "ec2:DescribeSubnets",
                      "ec2:DescribeTags",
                      "ec2:DescribeVpcAttribute",
                      "ec2:DescribeVpcs",
                      "ec2:DetachInternetGateway",
                      "ec2:DisassociateRouteTable",
                      "ec2:ModifyVpcAttribute",
                      "ec2:RunInstances",
                      "ec2:TerminateInstances",
                      "ecr:*",
                      "ecs:*",
                      "elasticloadbalancing:ApplySecurityGroupsToLoadBalancer",
                      "elasticloadbalancing:AttachLoadBalancerToSubnets",
                      "elasticloadbalancing:ConfigureHealthCheck",
                      "elasticloadbalancing:CreateLoadBalancer",
                      "elasticloadbalancing:DeleteLoadBalancer",
                      "elasticloadbalancing:DeleteLoadBalancerListeners",
                      "elasticloadbalancing:DeleteLoadBalancerPolicy",
                      "elasticloadbalancing:DeregisterInstancesFromLoadBalancer",
                      "elasticloadbalancing:DescribeInstanceHealth",
                      "elasticloadbalancing:DescribeLoadBalancerAttributes",
                      "elasticloadbalancing:DescribeLoadBalancerPolicies",
                      "elasticloadbalancing:DescribeLoadBalancerPolicyTypes",
                      "elasticloadbalancing:DescribeLoadBalancers",
                      "elasticloadbalancing:ModifyLoadBalancerAttributes",
                      "elasticloadbalancing:SetLoadBalancerPoliciesOfListener",
                      "iam:AttachRolePolicy",
                      "iam:CreateRole",
                      "iam:GetPolicy",
                      "iam:GetPolicyVersion",
                      "iam:GetRole",
                      "iam:ListAttachedRolePolicies",
                      "iam:ListInstanceProfiles",
                      "iam:ListRoles",
                      "iam:ListGroups",
                      "iam:ListUsers",
                      "iam:CreateInstanceProfile",
                      "iam:AddRoleToInstanceProfile",
                      "iam:ListInstanceProfilesForRole",

                      "iam:ListServerCertificates",
		                  "elasticloadbalancing:DescribeSSLPolicies"
                  ],
                  "Resource": "*"
                }
              ]
            }
            ```

1. **Create role to use with the ECS cluster to create** called `ecsInstanceRole` with this polciy, AmazonEC2ContainerServiceforEC2Role 

1. **Create a group and user allowed to deploy containers** (via updating services, creating tasks, not task definitions). 

    1. Create a group for the needed perimssions called `EcsDeploy`. Create a user and put it in.

    1. Add this policy the group: AmazonEC2ContainerRegistryFullAccess policy (allows ECR registry creation & pushing, but not management of ECS services, tasks, etc)

    1. Create and add a new policy called `EC2ContainerServiceDeploy` to the group with these permissions (deploying ECS only, but not setting up app via service/etc)

        ```json
        {
        "Version": "2012-10-17",
        "Statement": [
              {
                  "Sid": "Stmt1481041359000",
                  "Effect": "Allow",
                  "Action": [
                      "ecr:BatchCheckLayerAvailability",
                      "ecr:BatchGetImage",
                      "ecr:GetDownloadUrlForLayer",
                      "ecr:GetAuthorizationToken",
                      "ecs:DeregisterTaskDefinition",
                      "ecs:DescribeClusters",
                      "ecs:DescribeContainerInstances",
                      "ecs:DescribeServices",
                      "ecs:DescribeTaskDefinition",
                      "ecs:DescribeTasks",
                      "ecs:ListClusters",
                      "ecs:ListContainerInstances",
                      "ecs:ListServices",
                      "ecs:ListTaskDefinitionFamilies",
                      "ecs:ListTaskDefinitions",
                      "ecs:ListTasks",
                      "ecs:RegisterContainerInstance",
                      "ecs:RegisterTaskDefinition",
                      "ecs:RunTask",
                      "ecs:StartTask",
                      "ecs:StopTask",
                      "ecs:UpdateContainerAgent",
                      "ecs:UpdateService"
                  ],
                  "Resource": [
                      "*"
                  ]
              }
          ]
        }
        ```


    1. Create key pair for deployment auth

## *Part 2.* Setting up ECS for application deployment

This is the manual process to set up ECS and initially deploy an app. 

1. Log in as new IAM account to set up cluster.

1. Create cluster (or use existing and skip to the next step)
    1. Use `ecsInstanceRole`
    1. Note VPC, security group created for open ports

1. Create ECR registry named for the app (i.e. my-app).

1. Create AWS certificate in the ACM service to be used with the app's secured URLs via a load balancer.

1. Set up application load balancer:
    1. Create application load balancer.
    1. Use VPC used in sec. group for ECS EC2 instance, and that sec. group for the load balancer. Check that you can hit app via DNS name for created load balancer. 
    1. Set up subdomain for already registered domain name (if using a domain name registered with AWS, you can skip the last step):
        1. Create a hosted zone for the subdomain/etc you want to host using AWS Route 53.
        1. Add resource record sets for the new subdomain to Route 53 hosted zone.
        1. Update the DNS service for the parent domain by adding name server records for the subdomain provided in hosted zone record (it will take up to a few days for the DNS to propagate).
    1. Add listeners & targets for each task definition (app instance / environment) using AWS certificate.
        - Development: port https/44300 to container port 8080; http/8080 to 8080 (health check)
        - Staging: port https/44301 to container on port 8081; 8081 to 8081 (health check)
        - Production: https/443 to container on port 80; 80 to 80 (health check)
    1. Select 2 availability zones
    1. Make sure security group allows desired incoming ports

1. Prepare a task definition file for your app, including each container you have for the app. Then duplicate & update the task definition for each environment. You should end up with 1 task definition for each environment, with each task definition declaring all the containers it needs for the app, each pointing to the ECR app image, and each containing the appropriate settings for its environment (ports to expose, etc).
    - The task definitions are available in json files in this source code. You must add in the AWS account number.
    - Note: you can generate a task definition file from a Docker compose file with the ecs-cli, like this `ecs-cli compose -f *<docker compose file>* create`.

1. Create the task deinitions in ECS, by going into the ECS UI, going to Task Definitions and creating one. You can paste in the task definition json.

1. Create a service for each task definition (instance of the app - dev/stg/prod): go to ECS > Clusters > desired cluster and create a service. Choose the task definition, set tasks to 1, and set the minumum healthy count to 0 (so the task can be stopped before starting an updated one). Repeat for each environment. This should start your app instances.

1. For re-daploying the app for code changes, the process is to create a new task definition revision (with no changes, probably), then update the service to use this new one.
