# Installation Guide

## Cloud Native Distribution


## Getting Started with Kubernetes


## System Requirements for cloud deployments

!!!note
    For local deployments like `minikube` and `microk8s`  or cloud installations for demoing Gluu may set the resources to the minimum and hence can have `8GB RAM`, `4 CPU`, and `50GB disk` in total to run all services.
  
Please calculate the minimum required resources as per services deployed. The following table contains default recommended resources to start with. Depending on the use of each service the resources may be increased or decreased. 

|Service           | CPU Unit   |    RAM      |   Disk Space     | Processor Type | Required                                    |
|------------------|------------|-------------|------------------|----------------|---------------------------------------------|
|Auth-server            | 2.5        |    2.5GB    |   N/A            |  64 Bit        | Yes                                         |
|LDAP              | 1.5        |    2GB      |   10GB           |  64 Bit        | if using hybrid or ldap for persistence     |
|[Couchbase](#minimum-couchbase-system-requirements-for-cloud-deployments)         |    -       |      -      |      -           |     -          | If using hybrid or couchbase for persistence|
|config - job      | 0.5        |    0.5GB    |   N/A            |  64 Bit        | Yes on fresh installs                       |
|persistence - job | 0.5        |    0.5GB    |   N/A            |  64 Bit        | Yes on fresh installs                       |
|Admin-UI - coming-soon           | 1.0        |    1.0GB    |   N/A            |  64 Bit        | No                                          |
|nginx             | 1          |    1GB      |   N/A            |  64 Bit        | Yes if not ALB                              |


1. Configure cloud or local kubernetes cluster:

=== "EKS"
    ## Amazon Web Services (AWS) - EKS
      
    ### Setup Cluster
    
    -  Follow this [guide](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
     to install a cluster with worker nodes. Please make sure that you have all the `IAM` policies for the AWS user that will be creating the cluster and volumes.
    
    ### Requirements
    
    -   The above guide should also walk you through installing `kubectl` , `aws-iam-authenticator` and `aws cli` on the VM you will be managing your cluster and nodes from. Check to make sure.
    
            aws-iam-authenticator help
            aws-cli
            kubectl version
    
    - **Optional[alpha]:** If using Istio please [install](https://istio.io/latest/docs/setup/install/standalone-operator/) it prior to installing Gluu. You may choose to use any installation method Istio supports. If you have insalled istio ingress , a loadbalancer will have been created. Please save the address of loadblancer for use later during installation.

    !!!note
        Default  AWS deployment will install a classic load balancer with an `IP` that is not static. Don't worry about the `IP` changing. All pods will be updated automatically with our script when a change in the `IP` of the load balancer occurs. However, when deploying in production, **DO NOT** use our script. Instead, assign a CNAME record for the LoadBalancer DNS name, or use Amazon Route 53 to create a hosted zone. More details in this [AWS guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/using-domain-names-with-elb.html?icmpid=docs_elb_console).
      

=== "GKE"
    ## GCE (Google Cloud Engine) - GKE
    
    ### Setup Cluster

    1.  Install [gcloud](https://cloud.google.com/sdk/docs/quickstarts)
    
    1.  Install kubectl using `gcloud components install kubectl` command
    
    1.  Create cluster using a command such as the following example:
    
            gcloud container clusters create exploringgluu --num-nodes 2 --machine-type e2-highcpu-8 --zone us-west1-a
    
        where `CLUSTER_NAME` is the name you choose for the cluster and `ZONE_NAME` is the name of [zone](https://cloud.google.com/compute/docs/regions-zones/) where the cluster resources live in.
    
    1.  Configure `kubectl` to use the cluster:
    
            gcloud container clusters get-credentials CLUSTER_NAME --zone ZONE_NAME
    
        where `CLUSTER_NAME` is the name you choose for the cluster and `ZONE_NAME` is the name of [zone](https://cloud.google.com/compute/docs/regions-zones/) where the cluster resources live in.
    
        Afterwards run `kubectl cluster-info` to check whether `kubectl` is ready to interact with the cluster.
        
    1.  If a connection is not made to google consul using google account the call to the api will fail. Either connect to google consul using an associated google account and run any `kubectl` command like `kubectl get pod` or create a service account using a json key [file](https://cloud.google.com/docs/authentication/getting-started).
    
    - **Optional[alpha]:** If using Istio please [install](https://istio.io/latest/docs/setup/install/standalone-operator/) it prior to installing Gluu. You may choose to use any installation method Istio supports. If you have insalled istio ingress , a loadbalancer will have been created. Please save the ip of loadblancer for use later during installation.

    
=== "DOKS"
    ## DigitalOcean Kubernetes (DOKS)
    
    ### Setup Cluster
    
    -  Follow this [guide](https://www.digitalocean.com/docs/kubernetes/how-to/create-clusters/) to create digital ocean kubernetes service cluster and connect to it.

    - **Optional[alpha]:** If using Istio please [install](https://istio.io/latest/docs/setup/install/standalone-operator/) it prior to installing Gluu. You may choose to use any installation method Istio supports. If you have insalled istio ingress , a loadbalancer will have been created. Please save the ip of loadblancer for use later during installation.

=== "AKS"
    ## Azure - AKS
    
    !!!warning
        Pending
        
    ### Requirements
    
    -  Follow this [guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) to install Azure CLI on the VM that will be managing the cluster and nodes. Check to make sure.
    
    -  Follow this [section](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#create-a-resource-group) to create the resource group for the AKS setup.
    
    -  Follow this [section](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#create-aks-cluster) to create the AKS cluster
    
    -  Follow this [section](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#connect-to-the-cluster) to connect to the AKS cluster
    
    - **Optional[alpha]:** If using Istio please [install](https://istio.io/latest/docs/setup/install/standalone-operator/) it prior to installing Gluu. You may choose to use any installation method Istio supports. If you have insalled istio ingress , a loadbalancer will have been created. Please save the ip of loadblancer for use later during installation.

      
=== "Minikube"
    ## Minikube
    
    ### Requirements
    
    1. Install [minikube](https://minikube.sigs.k8s.io/docs/start/).
    
    1. Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
    
    1. Create cluster:
    
        ```bash
        minikube start
        ```
            
    1. Configure `kubectl` to use the cluster:
    
            kubectl config use-context minikube
            
    1. Enable ingress on minikube
    
        ```bash
        minikube addons enable ingress
        ```
        
    1. **Optional[alpha]:** If using Istio please [install](https://istio.io/latest/docs/setup/install/standalone-operator/) it prior to installing Gluu. You may choose to use any installation method Istio supports.Please note that at the moment Istio ingress is not supported with Minikube. 
    
=== "MicroK8s"
    ## MicroK8s
    
    ### Requirements
    
    1. Install [MicroK8s](https://microk8s.io/)
    
    1. Make sure all ports are open for [microk8s](https://microk8s.io/docs/)
    
    1. Enable `helm3`, `storage`, `ingress` and `dns`.
    
        ```bash
        sudo microk8s.enable helm3 storage ingress dns
        ```
        
    1. **Optional[alpha]:** If using Istio please enable it.  Please note that at the moment Istio ingress is not supported with Microk8s.
    
        ```bash
        sudo microk8s.enable istio
        ```   
      
2. Install Gluu using Helm
    
    1. **Optional if not using istio ingress:** Install [nginx-ingress](https://github.com/kubernetes/ingress-nginx) Helm [Chart](https://github.com/helm/charts/tree/master/stable/nginx-ingress).
    
       ```bash
       helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
       helm repo add stable https://charts.helm.sh/stable
       helm repo update
       helm install <nginx-release-name> ingress-nginx/ingress-nginx --namespace=<nginx-namespace>
       ```
    
    1.  - If the FQDN for gluu i.e `demoexample.gluu.org` is registered and globally resolvable, forward it to the loadbalancers address created in the previous step by nginx-ingress. A record can be added on most cloud providers to forward the domain to the loadbalancer. Forexample, on AWS assign a CNAME record for the LoadBalancer DNS name, or use Amazon Route 53 to create a hosted zone. More details in this [AWS guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/using-domain-names-with-elb.html?icmpid=docs_elb_console). Another example on [GCE](https://medium.com/@kungusamuel90/custom-domain-name-mapping-for-k8s-on-gcp-4dc263b2dabe).
    
        - If the FQDN is not registered acquire the loadbalancers ip if on **GCE**, or **Azure** using `kubectl get svc <release-name>-nginx-ingress-controller --output jsonpath='{.status.loadBalancer.ingress[0].ip}'` and if on **AWS** get the loadbalancers addresss using `kubectl -n ingress-nginx get svc ingress-nginx \--output jsonpath='{.status.loadBalancer.ingress[0].hostname}'`.
    
    1.  - If deploying on the cloud make sure to take a look at the helm cloud specific notes before continuing.
    
          * [EKS](#eks-helm-notes)
          * [GKE](#gke-helm-notes)
    
        - If deploying locally make sure to take a look at the helm specific notes bellow before continuing.
    
          * [Minikube](#minikube-helm-notes)
          * [MicroK8s](#microk8s-helm-notes)
    
    1.  **Optional:** If using couchbase as the persistence backend.
        
        1. Download [`pygluu-kubernetes.pyz`](https://github.com/GluuFederation/cloud-native-edition/releases). This package can be built [manually](#build-pygluu-kubernetespyz-manually).
        
        1. Download the couchbase [kubernetes](https://www.couchbase.com/downloads) operator package for linux and place it in the same directory as `pygluu-kubernetes.pyz`
    
        1.  Run:
        
           ```bash
           ./pygluu-kubernetes.pyz couchbase-install
           ```
           
        1. Open `settings.json` file generated from the previous step and copy over the values of `COUCHBASE_URL` and `COUCHBASE_USER`   to `global.gluuCouchbaseUrl` and `global.gluuCouchbaseUser` in `values.yaml` respectively. 
    
    1.  Create a namespace and install:    
       
       ```
       kubectl create ns gluu
       helm repo add jans https://janssenproject.github.io/jans-cloud-native/charts
       kubectl create ns jans
       # get ip of loadbalancer or node ip for microk8s/minikube. If on EKS get the address and use --set config.configmap.lbAddr="$Address"
       helm install jans-auth jans/jans -n jans --set global.lbIp="$ip" --devel
       ```
    Example commands: 
    
    Google
    
    ```
    ```
    
    Amazon Aurora
    
    ```
    ```
    ### EKS helm notes
    
    #### Required changes to the `values.yaml`
    
      Inside the global `values.yaml` change the marked keys with `CHANGE-THIS`  to the appropriate values :
    
      ```yaml
      #global values to be used across charts
      global:
        provisioner: kubernetes.io/aws-ebs #CHANGE-THIS
        lbAddr: "" #CHANGE-THIS to the address received in the previous step axx-109xx52.us-west-2.elb.amazonaws.com
        domain: demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
        isDomainRegistered: "false" # CHANGE-THIS  "true" or "false" to specify if the domain above is registered or not.
    
      nginx:
        ingress:
          enabled: true
          path: /
          hosts:
            - demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
          tls:
            - secretName: tls-certificate
              hosts:
                - demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
      ```    
    
      Tweak the optional [parameters](#configuration) in `values.yaml` to fit the setup needed.
    
    ### GKE helm notes
    
    #### Required changes to the `values.yaml`
    
      Inside the global `values.yaml` change the marked keys with `CHANGE-THIS`  to the appropriate values :
    
      ```yaml
      #global values to be used across charts
      global:
        provisioner: kubernetes.io/gce-pd #CHANGE-THIS
        lbAddr: ""
        domain: demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
        # Networking configs
        lbIp: "" #CHANGE-THIS  to the IP received from the previous step
        isDomainRegistered: "false" # CHANGE-THIS  "true" or "false" to specify if the domain above is registered or not.
      nginx:
        ingress:
          enabled: true
          path: /
          hosts:
            - demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
          tls:
            - secretName: tls-certificate
              hosts:
                - demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
      ```
    
      Tweak the optional [parameters](#configuration) in `values.yaml` to fit the setup needed.
    
    ### Minikube helm notes
    
    #### Required changes to the `values.yaml`
    
      Inside the global `values.yaml` change the marked keys with `CHANGE-THIS`  to the appopriate values :
    
      ```yaml
      #global values to be used across charts
      global:
        provisioner: k8s.io/minikube-hostpath #CHANGE-THIS
        lbAddr: ""
        domain: demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
        lbIp: "" #CHANGE-THIS  to the IP of minikube <minikube ip>
    
      nginx:
        ingress:
          enabled: true
          path: /
          hosts:
            - demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
          tls:
            - secretName: tls-certificate
              hosts:
                - demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
      ```
    
      Tweak the optional [parameters](#configuration) in `values.yaml` to fit the setup needed.
    
    - Map gluus FQDN at `/etc/hosts` file  to the minikube IP as shown below.
    
        ```bash
        ##
        # Host Database
        #
        # localhost is used to configure the loopback interface
        # when the system is booting.  Do not change this entry.
        ##
        192.168.99.100	demoexample.gluu.org #minikube IP and example domain
        127.0.0.1	localhost
        255.255.255.255	broadcasthost
        ::1             localhost
        ```
    
    ### Microk8s helm notes
      
    #### Required changes to the `values.yaml`
    
      Inside the global `values.yaml` change the marked keys with `CHANGE-THIS`  to the appopriate values :
    
      ```yaml
      #global values to be used across charts
      global:
        provisioner: microk8s.io/hostpath #CHANGE-THIS
        lbAddr: ""
        domain: demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
        lbIp: "" #CHANGE-THIS  to the IP of the microk8s vm
    
      nginx:
        ingress:
          enabled: true
          path: /
          hosts:
            - demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
          tls:
            - secretName: tls-certificate
              hosts:
                - demoexample.gluu.org #CHANGE-THIS to the FQDN used for Gluu
      ```
    
      Tweak the optional [parameteres](#configuration) in `values.yaml` to fit the setup needed.
    
    - Map gluus FQDN at `/etc/hosts` file  to the microk8s vm IP as shown below.
    
      ```bash
      ##
      # Host Database
      #
      # localhost is used to configure the loopback interface
      # when the system is booting.  Do not change this entry.
      ##
      192.168.99.100	demoexample.gluu.org #microk8s IP and example domain
      127.0.0.1	localhost
      255.255.255.255	broadcasthost
      ::1             localhost
      ```
      
    ### Uninstalling the Chart
    
    To uninstall/delete `my-release` deployment:
    
    `helm delete <my-release>`
    
    If during installation the release was not defined, release name is checked by running `$ helm ls` then deleted using the previous command and the default release name.
    


## Use Couchbase solely as the persistence layer

### Requirements
  - If you are installing on microk8s or minikube please ignore the below notes as a low resource `couchbase-cluster.yaml` will be applied automatically, however the VM being used must at least have 8GB RAM and 2 cpu available .
  
  - An `m5.xlarge` EKS cluster with 3 nodes at the minimum or `n2-standard-4` GKE cluster with 3 nodes. We advice contacting Gluu regarding production setups.

- [Install couchbase Operator](https://www.couchbase.com/downloads) linux version `2.1.0` is recommended but version `2.0.3` is also supported. Place the tar.gz file inside the same directory as the `pygluu-kubernetes.pyz`.

- A modified `couchbase/couchbase-cluster.yaml` will be generated but in production it is likely that this file will be modified.
  * To override the `couchbase-cluster.yaml` place the file inside `/couchbase` folder after running `./pygluu-kubernetes.pyz`. More information on the properties [couchbase-cluster.yaml](https://docs.couchbase.com/operator/1.2/couchbase-cluster-config.html).

!!!note
    Please note the `couchbase/couchbase-cluster.yaml` file must include at least three defined `spec.servers` with the labels `couchbase_services: index`, `couchbase_services: data` and `couchbase_services: analytics`

**If you wish to get started fast just change the values of `spec.servers.name` and `spec.servers.serverGroups` inside `couchbase/couchbase-cluster.yaml` to the zones of your EKS nodes and continue.**

- Run `./pygluu-kubernetes.pyz install-couchbase` and follow the prompts to install couchbase solely with Gluu.


## Use remote Couchbase as the persistence layer

- [Install couchbase](https://docs.couchbase.com/server/current/install/install-intro.html)

- Obtain the Public DNS or FQDN of the couchbase node.

- Head to the FQDN of the couchbase node to [setup](https://docs.couchbase.com/server/current/manage/manage-nodes/create-cluster.html) your couchbase cluster. When setting up please use the FQDN as the hostname of the new cluster.

- Couchbase URL base , user, and password will be needed for installation when running `pygluu-kubernetes.pyz`


### How to expand EBS volumes

1. Make sure the `StorageClass` used in your deployment has the `allowVolumeExpansion` set to true. If you have used our EBS volume deployment strategy then you will find that this property has already been set for you.

1. Edit your persistent volume claim using `kubectl edit pvc <claim-name> -n <namespace> ` and increase the value found for `storage:` to the value needed. Make sure the volumes expand by checking the `kubectl get pvc <claim-name> -n <namespace> `.

1. Restart the associated services



## Minimum Couchbase System Requirements for cloud deployments

The following are the minimum requirements that must be in place for a cloud deployment with [Couchbase](https://couchbase.com). This is not suitable for a production environment with minimal resources. Couchbase requires optimization and must be tested to suit the organizational needs. More information about how Gluu works with Couchbase can be found [here](../reference/persistence.md#couchbase).

 
| NAME                                     | # of nodes  | RAM(GiB) | Disk Space | CPU | Total RAM(GiB)                           | Total CPU |
| ---------------------------------------- | ----------- | -------  | ---------- | --- | ---------------------------------------- | --------- |
| Couchbase Index                          | 1           |  3       | 5Gi        | 1  | 3                                         | 1         |
| Couchbase Query                          | 1           |  -       | 5Gi        | 1  | -                                         | 1         |
| Couchbase Data                           | 1           |  3       | 5Gi        | 1  | 3                                         | 1         |
| Couchbase Search, Eventing and Analytics | 1           |  2       | 5Gi        | 1  | 2                                         | 1         |
| Grand Total                              |             | 7-8 GB (if query pod is allocated 1 GB)  | 20Gi | 4         |
