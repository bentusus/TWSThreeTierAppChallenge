# Overview  
This repository contains the code for `#TWSThreeTierAppChallenge` by [Train with Shubham](https://www.linkedin.com/posts/shubhamlondhe1996_twsthreetierappchallenge-trainwithshubham-activity-7151064404992679936-jsdZ?utm_source=share&utm_medium=member_desktop) with a twist. In this challenge we are going to create Kubernetes cluster on Microsoft Azure using [Azure Kubernetes Service](https://azure.microsoft.com/en-in/products/kubernetes-service). We are also going to create a private container registry using [Azure Container Registry](https://azure.microsoft.com/en-in/products/container-registry) and kubernetes cluster is exposed to the internet with the help of [Traefik](https://traefik.io/traefik/) ingress controller.


# Installations required on the development environment
- Microsoft Azure account
- Azure cli
- terraform 
- kubectl
- helm

# Steps
1. Login into the Azure account using azure cli
    ```
    az login
    ```
2. Create a new service principal  

    ```
    az ad sp create-for-rbac --skip-assignment
    ```

    Copy and save the information returned after executing the command successfully. Information will look something like:
    ```
    {
        "appId": "<app_if>",
        "displayName": "<display_name>",
        "password": "<password>",
        "tenant": "<tenant>"
    }
    ```

3. Get the service principal id(object id) using azure cli
    ```
    az ad sp show --id <appId_from_above_step> --query "id"
    ```
    Note the retured principal id as well.

4. In the terraform folder create a new file `terraform.tfvars` and paste the below code to initialise the variable values:
    ```
    resource_group_name = "tws_deployment_RG"
    location            = "centralindia"
    cluster_name        = "my-aks-cluster"
    kubernetes_version  = "1.26.10"
    system_node_count   = 3
    acr_name            = "twsChallengeACRVishal"
    appId               = "<appId_from_step_2>"
    principalid         = "<principalId_from_step_3>"
    password            = "<password_from_step_2>"
    dns_prefix          = "aks-dns-prefix-k8s"
    ```
    In `appId`, `principalid`, `password` insert data from the previous steps

5. Run the terraform commands to create a new AKS cluster and ACR.
    ```
    terraform init
    terraform fmt
    terraform plan
    # If the plan looks fine then go ahead
    terraform apply -auto-approve
    ```
    Make note of the outputs after all the resources are created. Especially `acr_login_server`, `acr_username` and `acr_password`.  
    To see `acr_password`, use the command 
    ```
    terraform output acr_password
    ```

6. After provisioning, Retrieve access credentials and automatically configure `kubectl`
    ```
    az aks get-credentials --resource-group $(terraform output -raw resource_group_name) --name $(terraform output -raw kubernetes_cluster_name)
    ```

7. Login into the private container registry hosted on Azure
    ```
    docker login <acr_login_server>
    ```
    Change `<acr_login_server>` to your login server.  
    Provide the `acr_username` and `acr_password` when prompted.

8. Build and push docker images for `backend` and `frontend`.
    ```
    # build and push backend
    cd backend
    docker image build -t <acr_login_server>/backend:v1 ./
    docker image push <acr_login_server>/backend:v1


    # build and push frontend  
    cd ../frontend
    docker image build -t <acr_login_server>/frontend:v1 ./
    docker image push <acr_login_server>/frontend:v1
    ```
    Change the `acr_login_server` to your login server name

9. Change the env. value in `k8s_manifests/frontend-deployment.yaml` to `http://app.20.207.122.241.nip.io/api/tasks`   
and host in  `traefik-ingress-controller/ingress.yaml` to `app.<public_ip_kubernets_lb>.nip.io`.  
If you own a custom domain, then create a subdomain and change `app.20.207.122.241.nip.io` to your sub domain name.  
Learn more about [nip.io](https://nip.io/)


10. Run the K8S manifests to create kubernetes deployment and services.
    ```
    kubectl create workspace workshop
    kubectl apply -f <all_the_yaml_files> -n workshop
    ```
    In `<all_the_yaml_files>`, provide all the yaml files from the `k8s_manifests` folder and `traefik-ingress-controller` folder.

11. Install [Traefik](https://traefik.io/traefik/) using helm.
    ```
    helm repo add traefik https://helm.traefik.io/traefik
    helm repo update
    kubectl create namespace traefik
    helm install traefik traefik/traefik -n traefik
    ```

12. Application is deployed on the AKS cluster. To access it run `http://app.20.207.122.241.nip.io/` or the sub domain on your custom domain provided during step 9