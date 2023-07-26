# Demo for Deploying to EKS Cluster from Jenkins

## Install Kubectl Inside the Docker container

We want to isntall Kubectl inside the container that Jenkins running. The instructions are writte [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

    docker exec -u 0 -it {container-id} bash
  
## Install AWS IAM Authenticator

For instructions navigate [here](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html).

    curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.5.9/aws-iam-authenticator_0.5.9_linux_amd64
  
    chmod +x ./aws-iam-authenticator
    mv ./aws-iam-authenticator /usr/local/bin
    
    # Test the installation
    aws-iam-authenticator

We can now exit the docker container.

    exit
    
## Create Kube Config File
We don't have any editor inside the Docker container. We are going to create the config file on the server itself and copy it to the container.
The instructions can be accessed [here](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)
    
    # In our local machine:
    export region_code=eu-central-1
    export cluster_name=demo-cluster
    export account_id=0000000000
    
    cluster_endpoint=$(aws eks describe-cluster \
        --region $region_code \
        --name $cluster_name \
        --query "cluster.endpoint" \
        --output text)
    
    # Continue with:
    
    #!/bin/bash
    read -r -d '' KUBECONFIG <<EOF
    apiVersion: v1
    clusters:
    - cluster:
        server: $cluster_endpoint
        certificate-authority-data: $certificate_data
      name: arn:aws:eks:$region_code:$account_id:cluster/$cluster_name
    contexts:
    - context:
        cluster: arn:aws:eks:$region_code:$account_id:cluster/$cluster_name
        user: arn:aws:eks:$region_code:$account_id:cluster/$cluster_name
      name: arn:aws:eks:$region_code:$account_id:cluster/$cluster_name
    current-context: arn:aws:eks:$region_code:$account_id:cluster/$cluster_name
    kind: Config
    preferences: {}
    users:
    - name: arn:aws:eks:$region_code:$account_id:cluster/$cluster_name
      user:
        exec:
          apiVersion: client.authentication.k8s.io/v1beta1
          command: aws-iam-authenticator
          args:
            - "token"
            - "-i"
            - "$cluster_name"
            - "--role"
            - "arn:aws:iam::$account_id:role/eksctl-demo-cluster-cluster-ServiceRole-N0WJU5IFOGX0"
    EOF
    echo "${KUBECONFIG}" > ~/config

    
    # Copy the generate file to the server where Jenkins is running on a docker container
    scp config root@164.90.231.208:/root
    
    # On the server where the container is running
    docker exec -it {container-id-of-jenkins} bash
    cd ~
    
    # This gives us the path that the Kube config file will end up in. (/var/jenkins_home)
    pwd
    
    mkdir .kube
    exit
    docker cp config {container-id}:/var/jenkins_home/.kube/
     
    # We can now confirm the copy and check if the config file can be found in the .kube directory
    docker exec -it {container-id} bash
    
## Create AWS Credentials

It is best practice to create a new user for Jenkins on AWS to restrict the access. In this exercise we use the admin user for the sake of simplicity. We create a new credential in Jenkins (in this case in our multibranch pipeline)

The key can be accessed under ~/.aws/credentials

aws_access_key_id:

![image](https://user-images.githubusercontent.com/18715119/234788969-79c90687-781a-4377-97ba-999177ea179f.png)

We also create another one for aws_secret_access_key:

![image](https://user-images.githubusercontent.com/18715119/234789293-d297dd4d-2f56-437a-8654-cbc4ef7da944.png)

## Configure Jenkinsfile to deploy to EKS

The configuration steps are added to the Jenkinsfile.
