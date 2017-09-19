# ECS command reference 

### Deregister container instance 
    `
    aws --region us-east-1 ecs deregister-container-instance --cluster defic-cluster --container-instance container_instance_id --force
    `
## Create ECS environment with ecs-cli 

1. Install ecs-cli

1. Set region: 
    `
    ecs-cli configure -c <cluster name> --region <region> 
    `

1. Create cluster of ECS container instances to launch containers on: 
    `
    ecs-cli up --keypair id_rsa --capability-iam --size 2 --instance-type t2.medium
    `

# Manually deploy to ECR, then ECS

## Push to EC2 Container Registry (ECR) with aws-cli:

1. Log into ECR, build an image and put it in:

    ```
    aws ecr get-login --region us-east-1 

    *Then run the docker login command given back, with token*

    docker build -t deficiency-ecr .

    docker tag deficiency-ecr:latest <ACCT_ID>.dkr.ecr.us-east-1.amazonaws.com/deficiency-ecr:latest

    docker push <ACCT_ID>.dkr.ecr.us-east-1.amazonaws.com/deficiency-ecr:latest
    ```

## Deploy with ecs-cli (generates a task definition from a compose file)

    `
    ecs-cli compose --verbose --file ./docker-compose-release.yml up 
    `

    - Check container status: `ecs-cli ps`

    - Check ECR image(s): `aws ecr list-images --repository-name deficiency-ecr`


## Notes 

- Login to ECS/EC2 instance: `ssh -i <path-to-key> <instance URL>`