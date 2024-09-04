# Azure-AKS-ARC-Setup

Documentation for bringing up an Azure Kubernetes cluster integrated with GitHub Actions Runner Controller for IREE Project

## Step 1: Create Azure Kubernetes Service

Search for Kubernetes Service in the top search bar in Azure Portal. Once in, now click on Create -> Kubernetes Cluster. 
Choose your resource group and cluster name and proceed with default options for Basics.
Next, you should be in the Node Pools section. You will see two node pools (userpool and agentpool).
The agentpool is in System mode and is designed to host the critical system pods that Kubernetes needs to operate.
The userpool is the one we care about. It is in user mode and used is designed to host the applications and workloads that we deploy to our Kubernetes cluster.
The userpool is where our github actions jobs will be dispatched. The default VM being used for these nodes is Standard_D8ds_v5. This only has 8 cores, and we need more for our IREE project.
Delete the userpool and create a new pool. For Node Size, select `Standard F48s v2`. Also, make sure to select the `Autoscale` option.
I went with this VM because out of all the 48 core ones, it is the only one that is optimized for compute (performance), lacking in memory/storage (which we don't need), and costs $1.67/hr.

<img width="853" alt="image" src="https://github.com/user-attachments/assets/08076afe-772c-4702-a3e2-39066494b2b9">



For the rest of the cluster creation options you can choose the default.

## Step 2: Login to your Cluster

Now, to configure the cluster and all the services you need to connect to the cluster.
You can do this in your own local dev environment (just make sure you have kube, helm, and azure cli installed)
I just use the cloud shell that Azure provides. (Click `Connect` and then `Cloud Shell`)
Run these two commands to connect to your cluster:
```
az account set --subscription <your_subscription_number>
az aks get-credentials --resource-group <resource_group_name> --name <cluster_name> --overwrite-existing
```

# Latest GHA Runner Scale Set Instructions

# Step 3: Install GHA Scale Set controller and listener

```
helm upgrade --install "azure-linux-scale"     --namespace "arc-runners"     --create-namespace     --set githubConfigUrl="https://github.com/iree-org"     --set githubConfigSecret.github_token="<your_PAT_token>"     oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set -f values.yaml
```

Please use the values.yaml file from latest-config-files folder in this repo for the above command.
The "azure-linux-scale" is the installation name and becomes the label you use for "runs-on:", so configure it however you see fit.

The values.yaml uses a custom docker image that is currently hosted in this repo: https://github.com/saienduri/docker-images/tree/main.
It also has CI configured to build and publish the image.
We need a custom image that builds off the gha scale set runner docker image provided, so that it has the deps that we require for CI and works on all our workflows.
Other things we configure in the values.yaml file is min runners = 3 and max runners = 30 for scaling.
The scaling setup is basically the same as the legacy documentation below, so please refer to that for further details.
Also, docker in docker is setup, so in our github workflows we can specify images to use if we want (iree uses cpubuilder_ubuntu_jammy image for example), but as done in iree-turbine, we can just run workflows using the preconfigured custom image here without further setup and that works too.

Now, change your workflows appropriately to match the label set with the installation name above and enjoy the AKS + ARC magic :)

# Legacy ARC Instructions (still works)

## Step 3: Install Cert Manager

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.15.3 --set crds.enabled=true
```

Cert-Manager is a Kubernetes add-on that automates the management and issuance of TLS (Transport Layer Security) certificates.
This is used for security reasons.

## Step 4: Install Github ARC and Authenticate

I do this using a personal token. So, if you don't have one, create a github token with these permissions:

```
repo (all)
admin:org (all) (mandatory for organization-wide runner)
admin:enterprise (all) (mandatory for enterprise-wide runner)
admin:public_key - read:public_key
admin:repo_hook - read:repo_hook
admin:org_hook
notifications
workflow
```

We will also be adding a webhook server as part of installing the actions-runner-controller, so we need to create a secret for the server to authenticate the github webhooks coming in.

```
kubectl create namespace actions-runner-system
kubectl create secret generic github-selfhosted-webhook-token -n actions-runner-system --from-literal=SELFHOSTED_GITHUB_WEBHOOK_SECRET_TOKEN=<your_webhook_secret>
```

Then, use the following command to install the github ARC

```
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm repo update
helm upgrade --install --namespace actions-runner-system --set=authSecret.create=true --set=authSecret.github_token="<your_token>"  --wait actions-runner-controller actions-runner-controller/actions-runner-controller -f runner-controller.yaml
```

The yaml file used above configures the actions runner controller service and the webhook server. I've added the yaml file I used (`runner-controller.yaml`) to this repo.
Here we tell it to configure a bunch of things for the runner controller, and we give it a docker image to use.
I've set it up to use `summerwind/actions-runner:ubuntu-22.04` which is the latest one provided by the github actions controller with dind enabled.
This works fine for us and passes all iree-turbine jobs (with no docker) and the iree jobs (these use multiple docker images and work through dind)

## Step 5: Configure GitHub Webhooks

I've set this up to use webhooks to drive the overall scaling of our cluster.
This scaling is performed based on the number of webhook events received from GitHub.
Here's an image on how that overall process works:

![image](https://github.com/user-attachments/assets/b11266c5-0c80-4a34-aa18-19a4da255965)


To configure this, first we need to expose the github-webhook server created above to the public, so it can receive from GitHub API.
To do this, get the current configuration if the server using this command:
`kubectl get svc actions-runner-controller-github-webhook-server -n actions-runner-system -o yaml > current-config.yaml`

Then, open up current-config.yaml and change spec type from `ClusterIP` to `LoadBalancer` in the yaml file and also delete the following lines which aren't neccesary after the switch.
Also change `http` to `https` in the config.
```
clusterIP: 10.0.11.74
  clusterIPs:
  - 10.0.11.74
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
```
TODO(saienduri): Find a way to just configure it with a load balancer initially (just webhook server, not the service)

Then, to actually update the service to use the updated config:
```
kubectl apply -f current-config.yaml
```

Now that the server and webhook secret have been configured, you can go to the github org/repo to set up the github side of things.
Go to "Settings" -> "Webhooks".
Create a new webhook with address `http://<external-ip>/webhooks` and the content type as `application/json`.
Then in the secret section add the secret that we added earlier.
For events, you can pick "Let me select individual events" and then choose push, workflow, and workflow jobs.
If you don't know the external IP of the webhook server you can run:
`kubectl get svc -n actions-runner-system`

<img width="566" alt="image" src="https://github.com/user-attachments/assets/76e5d247-c5dd-4aef-aba1-374b789ce7f8">


## Step 6: Deploy the Runners

Here, we deploy the runners.
Specifically, we tell the actions runner controller how much resources we need (45 cores, 50 GB).
We also give it a runner label that we use in the actual workflow `runs-on:` (I use azure-linux in the yaml)
You can use the yaml in this repo (runner-deployment.yaml) in the following command:

`kubectl apply -f runner-deployment.yaml`

## Step 7: Configure HRA

This is to configure GitHub Actions Runner Controller's HorizontalRunnerAutoscaler (HRA).
With the GitHub Actions Runner Controller in a Kubernetes cluster, each runner corresponds to a single container within a pod, and each pod only runs one runner.
This particular design of the Actions Runner Controller makes sure that each runner operates in its own isolated environment, for the best security of concurrent CI jobs running.
So, you can think of HRA as a specialized version of HPA, and we don't need it in the GitHub ARC context.
Here, we tell HRA to scale the GitHub Actions runners based on the webhooks we configured earlier.
Specifically, we trigger an autoscale everytime there is a webhook event for a workflow, so a runner will be requested.
It will also downscale appropriately.
You can use the yaml in this repo (horizontal-scale.yaml) for the following command:

`kubectl apply -f horizontal-scale.yaml`

Basically there are two levels of autoscaling.
HRA adjusts the number of pods to meet the runner demand.
If the number of pods increases beyond the capacity of the current nodes, the Cluster Autoscaler (the thing we setup at the very start) steps in to scale up the node pool, adding more nodes to provide the necessary resources for the additional pods.


Now, change your workflows appropriately to match the labels set in the runner-deployment.yaml and enjoy the AKS + ARC magic :)











