# Example for Continuos development with AWS ECS.

## Services implemented

* ECS.
* ECR.
* CodeBuild.
* CodeCommit.
* CodePipeline.
  

### RUN LOCAL
1. Install Docker in your computer for local test.
2. Clone this repo.
3. Move to the folder repo.
4. Proceed to build the docker image from your DockerFile.
   ```
   docker build -t demo .
   ```
5. Proceed to run docker image to verify that image was created correctly.
   ```
   docker run -p 80:80 demo
   ```
6. If you are running Docker locally, point your browser to http://localhost/.

### SEND IMAGE TO ECR

1. Proceed to create a new repository to this test. After has been created proceed to modified the policy for this repository.
   
   The current policy must be defined to allow access from CodeBuild.

   ```json
   {
    "Version": "2008-10-17",
    "Statement": [
        {
        "Sid": "CodeBuildAccess",
        "Effect": "Allow",
        "Principal": {
            "Service": "codebuild.amazonaws.com"
        },
        "Action": [
            "ecr:BatchCheckLayerAvailability",
            "ecr:BatchGetImage",
            "ecr:GetDownloadUrlForLayer"
        ]
        }
    ]
    }
   ```
    Save policy.

2. After is necessary push the image created previously. For this:
   
    1. Retrieve the login command to use to authenticate your Docker client to your registry.
(Use the AWS CLI): 
    ```
    $(aws ecr get-login --no-include-email --region [REGION])
    ```
    Example
    ```
    $(aws ecr get-login --no-include-email --region us-east-1)
    ```
   2. After login, tag your image created previously so you can push the image to this repository. Remember that the label of the image must be the same arn of the previously created repository:
    ```
    docker tag [NAME-IMAGE] [ID-ACCOUNT].dkr.ecr.[REGION].amazonaws.com/[NAME-REPO]:latest
    ```
    Example

    ```
    docker tag demo 374387630610.dkr.ecr.us-east-1.amazonaws.com/demo:latest
    ```
   3. Push this image to your newly created AWS repository:
    ```
    docker push [NAME-IMAGE] [ID-ACCOUNT].dkr.ecr.[REGION].amazonaws.com/[NAME-REPO]:latest
    ```
    4. Confirm the image push in the repository ECR.

3. Repository to manage the changes - **CodeCommit**.
   1. Proceed to create a new repository in CodeCommit.
   2. Clone the new repository in your local enviroment and copy all files from this repo.
   3. push files to repository en CodeCommit created previously.
   4. Check files in Codecommit repository.

### CodeBuild

   1. Create a new project in CodeBuild. For values:
        * Project Name.
        * Description (optional).
        * Source Provider (**select AWS CodeCommit**).
        * Repository (**select repository created previously**).
        * Environment image (**select Managed image**).
        * Operating system (**select Ubuntu, runtime Docker, runtime version docker:17.09.0 and image version choice Always use the latest image for this runtime version**).
        * Service role (**select New service role, define a name that refers to the service. Later it will be modified**).
        * Build specifications (**select Use a buildspec file**).
        
        **All other values maintaining default**
        
        * Create build project. (if show problems with the policy **CodeBuildBasePolicy-demo-us-east-1**. Please, delete this policy and retry)

   2. Go to service IAM and proceed to associate the next permissión **AmazonEC2ContainerRegistryPowerUser** to the role create before:

### ECS

1. Let's create an ELB to allow HA in cluster.
    * Create a ELB type ALB:
        * Name
        * Scheme (**internet-facing**).
        * IP addresstype (**ipv4**).
        * Listener (**Protocol HTTP, Load balancer port 80**).
        * VPC (**selected VPC**).
        * AZ (**AZ public for ELB, almost 2**).
        * Security group (**create SG to Allow 80 from 0.0.0.0/0**).
        * Define target:
          * Name
          * Target type (**instance**).
          * protocol (**http**).
          * port (**80**).
          * Health checks protocol (**http**).
          * Path (**/**).
          * All other values in default.
          * For Register target. **not associate any element, the ECS service will take care of that activity.**. Next.
          * Review and Create.

2. Create a new Cluster from GUI.For this case select **EC2 Linux + Networking**. With this values:
    * Name
    * Previsioning Model (**On-Demand Instance**).
    * EC2 instance type (**your choice**).
    * Number of instances (**for this test 1**).
    * EC2 Ami Id (**is selected automatic**).
    * EBS storage (GiB) (**is selected automatic**).
    * Key pair (**select key**).
    * VPC (**select same VPC of ELB**).
    * Subnet (**select subnet**).
    * Security Group (**create new SG with rules for default**).
    * Container instance IAM role (**choice ecsInstanceRole**).
    * Created new Cluster.
  
3. it is necessary to allow communication between the previously created ELB and the instance created by the ECS service. for this we must modify the associated SG in the instance and allow All access from the SG that is associated with the ALB. For this Example:

ELB (sg-03f791d10a2e4266x)

| Type | Protocol | Port Range | Source    | Description |
|------|----------|------------|-----------|-------------|
| HTTP | TCP      | 80         | 0.0.0.0/0 |             |


Instance EC2 (ECS)

| Type    | Protocol | Port Range | Source               | Description |
|---------|----------|------------|----------------------|-------------|
| ALL TCP | TCP      | 0 - 65535  | sg-03f791d10a2e4266x | SG from ALB |


**With this configurations only allow access to the ELB.**
 
1. Go to created a new task for the cluster. The file **simple-app-task-def.json** contains the task to create. Proceed to open for change some values: 
    ```json
    {
        "family": "[NAME-OF-CLUSTER]",
        "volumes": [
            {
                "name": "my-vol",
                "host": {}
            }
        ],
        "containerDefinitions": [
            {
                "environment": [],
                "name": "[NAME-OF-CLUSTER]",
                "image": "[NAME-IMAGE] [ID-ACCOUNT].dkr.ecr.[REGION].amazonaws.com/[NAME-REPO]:latest",
                "cpu": 10,
                "memory": 500,
                "portMappings": [
                    {
                        "containerPort": 80,
                        "protocol": "tcp"
                    }
                ],
                "mountPoints": [
                    {
                        "sourceVolume": "my-vol",
                        "containerPath": "/var/www/my-vol"
                    }
                ],
                "entryPoint": [
                    "/usr/sbin/apache2",
                    "-D",
                    "FOREGROUND"
                ],
                "essential": true
            },
            {
                "name": "busybox",
                "image": "busybox",
                "cpu": 10,
                "memory": 500,
                "volumesFrom": [
                {
                "sourceContainer": "[NAME-OF-CLUSTER]"
                }
                ],
                "entryPoint": [
                    "sh",
                    "-c"
                ],
                "command": [
                    "/bin/sh -c \"while true; do /bin/date > /var/www/my-vol/date; sleep 1; done\""
                ],
                "essential": false
            }
        ]
    }
    ```
2. Before saved changes. Proceed to create new task from CLI AWS:
    ```
    aws ecs register-task-definition --family [NAME-OF-TASK] --cli-input-json file://[PATH-OF-JSON]
    ```
    Example
    ```
    aws ecs register-task-definition --family demo  --cli-input-json file://simple-app-task-def.json
    ```
3. Validate new task in console of ECS.
4. Create new Service to the cluster. With this values:
   1. Configure service
       * Launch Type (**EC2**).
       * Task definition (**created previously**).
       * Cluster (**created previously**).
       * Service name.
       * Number of task (**1 for this example**).
       * Other values in default for first page. Next step.
   2. Configure network
       * Health check grace period (**60 for example**).
       * Load balancer type (**select Application Load Balancer**).
       * Service IAM role (**select AWSServiceRoleForECS**).
       * Load balancer name (**created previously**).
       * Container name : port (**select add to load balancer and configure**)
         * Production listener port (**80:HTTP**).
         * Target group name (**created previously**).
         * Other values in default. Next step.
   3. Set Auto Scaling (optional)
         * Service Auto Scaling (**select Do not adjust the service’s desired count**).
         * Next step, review and create service.
   4. Go to the task in cluster and view the status in current task.
   5. Validate that the service is running, use the DNS balancer with the web browser.
   



Directions on how to launch this sample app on Amazon ECS can be found in the documentation: 
[Docker basics](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html).

### CodePipeline


1. Create pipeline:
   1. Choose pipeline settings 
      * Name.
      * Service role (**select new service role o select one**).
      * Artifact store (**default location**).
   2. Add source stage
      * Source provider (**AWS CodeCommit**).
      * Repository name (**select the previously created**).
      * Branch name (**select master if is default**).
      * Change detection options (**Amazon CloudWatch Events (recommended)**).
   3. Add build stage
      1. Build provider (**select AWS CodeBuild**).
      2. Project name (**select the previously created**).
   4. Add deploy stage
      1. Deploy provider (**select AWS ECS**).
      2. Cluster name (**select cluster previously created**).
      3. Service name (**select service previously created**).
   5. Review and create pipeline.

2. You will notice that after the creation of the pipeline, the first process will begin. This will take a few minutes to complete.
3. Make changes to the repository, any file, example (index.php). push repository and validate the execution of the pipeline.
   
### Documentation

[Continuous Deployment with AWS CodePipeline.](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-cd-pipeline.html)

[Create a Pipeline with an Amazon ECR Source and ECS-to-CodeDeploy Deployment.](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-ecs-ecr-codedeploy.html)

[Setting Up for AWS CodeCommit.](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up.html?icmpid=docs_acc_console_connect_np#setting-up-compat)