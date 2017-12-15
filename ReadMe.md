

# Setting up JupyterHub on Microsoft Azure Container Service (ACS) #

Note
These are updated from the alpha docs at 
[https://zero-to-jupyterhub.readthedocs.io/en/v0.4-doc/create-k8s-cluster.html#setting-up-kubernetes-on-microsoft-azure-container-service-acs](https://zero-to-jupyterhub.readthedocs.io/en/v0.4-doc/create-k8s-cluster.html#setting-up-kubernetes-on-microsoft-azure-container-service-acs) This includes problems I ran into when developing this build.

Todo for this doc - add custom docker image update, expand authentication section, and possible 'instant Azure' button.

These commands are best executed in Bash for Windows. You can do it in a regular command line but you may run into some challenges installing Helm for Windows. The build is rather smooth for Bash with the updated Azure CLI install instructions below.

## Setting up the Kubernetes Cluster on ACS ##


1.	Install and initialize the Azure command-line tools, which send commands to Azure and let you do things like create and delete clusters.

Instructions for doing this are here: [https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)


2.	Authenticate the az tool so it may access your Azure account: **az login**
3.	Specify a Azure resource group, and create one if it doesn’t already exist:

    export RESOURCE_GROUP=<YOUR_RESOURCE_GROUP>
    export LOCATION=<YOUR_LOCATION>
    az group create --name=${RESOURCE_GROUP} --location=${LOCATION}

where:
•	--name specifies your Azure resource group. If a group doesn’t exist, az will create it for you.
•	--location specifies which computer center to use. To reduce latency, choose a zone closest to whoever is sending the commands. (View available zones via **az account list-locations**.)
	

Install kubectl, a tool for controlling Kubernetes:
**az acs kubernetes install-cli**

(note that if you are in bash, you have to **sudo az acs kubernetes install-cli**.)

4.	Create a Kubernetes cluster on Azure, by typing in the following commands:
    export CLUSTER_NAME=<YOUR_CLUSTER_NAME>
    export DNS_PREFIX=<YOUR_PREFIX>
    	az acs create --orchestrator-type=kubernetes \
     --resource-group=${RESOURCE_GROUP} \
      --name=${CLUSTER_NAME} \
     --dns-prefix=${DNS_PREFIX}
    

5.	Authenticate kubectl:
    az acs kubernetes get-credentials \
    --resource-group=${RESOURCE_GROUP} \
    --name=${CLUSTER_NAME}
where:
•	--resource-group specifies your Azure resource group.
•	--name is your ACS cluster name.
•	--dns-prefix is the domain name prefix for the cluster.


6.	To test if your cluster is initialized, run: **kubectl get node**

The response should list three running nodes.

## Setting up Helm ##

Helm, the package manager for Kubernetes, is a useful tool to install, upgrade and manage applications on a Kubernetes cluster. We will be using Helm to install and manage JupyterHub on our cluster.


The simplest way to install helm is to run Helm’s installer script at a terminal:
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash


Alternative methods for helm installation exist if you prefer to install without using the script.

**Initialization**

In Azure ACS, you don’t need to do other initialization for your cluster. 
You will need to update helm to make sure that both the server and client are running the same versions.
    helm init
    helm init --upgrade
 
Verify
You can verify that you have the correct version and that it installed properly by running:
**helm version**
It should provide output like
    Client: &version.Version{SemVer:"v2.7.2", GitCommit:"08c1144f5eb3e3b636d9775617287cc26e53dba4", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.7.2", GitCommit:"08c1144f5eb3e3b636d9775617287cc26e53dba4", GitTreeState:"clean"}

Make sure your client and server versions are the same version.



## Prepare configuration file ##

This step prepares a configuration file (config file). We will use the YAML file format to specify JupyterHub’s configuration. I've added the sample file to this Repo.

It’s important to save the config file in a safe place. The config file is needed for future changes to JupyterHub’s settings.

For the following steps, use your favorite code editor. I used Notepad because it added no additional formatting and seemed to be the best choice.

1.	Create a file called config.yaml. 
2.	Create two random hex strings to use as security tokens. Run these two commands (they’re the same command but run them twice) in a terminal:
    openssl rand -hex 32
    openssl rand -hex 32
Copy the output each time, we’ll use these hex strings in the next step.

3.	Insert these lines into the config.yaml file (provided on this repo). When editing YAML files, use straight quotes and spaces and avoid using curly quotes or tabs. Substitute each occurrence of RANDOM_STRING_N below with the output of openssl rand -hex 32. The random hex strings are tokens that will be used to secure your JupyterHub instance (make sure that you keep the quotation marks):



4.	Save the config.yaml file.


## Install JupyterHub ##

1.	Let’s add the JupyterHub helm repository to your helm, so you can install JupyterHub from it. This makes it easy to refer to the JupyterHub chart without having to use a long URL each time.
    helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
    helm repo update

This should show output like:
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "jupyterhub" chart repository
Update Complete. ⎈ Happy Helming!⎈

2.	Now you can install the chart! Run this command from the directory that contains the config.yaml file to spin up JupyterHub:
    helm install jupyterhub/jupyterhub \
    --version=v0.4 \
    --name=<YOUR-RELEASE-NAME> \
     --namespace=<YOUR-NAMESPACE> \
     -f config.yaml \
    --set prePuller.enabled=false
    
where:
- --name is an identifier used by helm to refer to this deployment. You need it when you are changing the configuration of this install or deleting it. Use something descriptive that you will easily remember. For a class called data8 you might wish set the name to data8-jupyterhub. In the future you can find out the name by using helm list.
- --namespace is an identifier used by Kubernetes (among other things) to identify a particular application that might be running on a single Kubernetes cluster. You can install many applications into the same Kubernetes cluster, and each instance of an application is usually separated by being in its own namespace. You’ll need the namespace identifier for performing any commands with kubectl.

We recommend providing the same value to --name and --namespace for now to avoid too much confusion, but advanced users of Kubernetes and helm should feel free to use different values.

Right now there is a bug with the pre-puller in helm, so you have to skip it with the line **--set prePuller.enabled=false**

Note
If you get a release named <YOUR-RELEASE-NAME> already exists error, then you should delete the release by running **helm delete --purge <YOUR-RELEASE-NAME>**. Then reinstall by repeating this step. If it persists, also do **kubectl delete <YOUR-NAMESPACE>** and try again.

If you’re pulling from a large Docker image you may get a Error: timed out waiting for the condition error, add a **--timeout=SOME-LARGE-NUMBER** parameter to the helm install command.

While Step 1 is running, you can see the pods being created by entering in a different terminal:
    kubectl --namespace=<YOUR_NAMESPACE> get pod

Wait for the hub and proxy pod to begin running.

You can find the IP to use for accessing the JupyterHub with:

    kubectl --namespace=<YOUR_NAMESPACE> get svc

The external IP for the proxy-public service should be accessible in a minute or two.

Note
If the IP for proxy-public is too long to fit into the window, you can find the longer version by calling:
kubectl --namespace=<YOUR_NAMESPACE> describe svc proxy-public


To use JupyterHub, enter the external IP for the proxy-public service in to a browser. JupyterHub starts out with running with a default dummy authenticator so entering any username and password combination will let you enter the hub.






