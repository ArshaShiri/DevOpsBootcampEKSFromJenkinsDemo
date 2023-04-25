# Demo for Deploying EKS Cluster from Jenkins

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
    
    # Craete the config file
    vim config
    
Add the follwowing blueprint from the link above:

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
            # - "- --role"
            # - "arn:aws:iam::$account_id:role/my-role"
          # env:
            # - name: "AWS_PROFILE"
            #   value: "aws-profile"

