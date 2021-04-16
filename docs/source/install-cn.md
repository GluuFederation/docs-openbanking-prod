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
|config - job      | 0.5        |    0.5GB    |   N/A            |  64 Bit        | Yes on fresh installs                       |
|persistence - job | 0.5        |    0.5GB    |   N/A            |  64 Bit        | Yes on fresh installs                       |
|nginx             | 1          |    1GB      |   N/A            |  64 Bit        | Yes if not ALB or Istio                             |


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
      
    
=== "Quick start with Microk8s"
    ## MicroK8s
    
    ### Requirements
    
    1. Create a fresh Ubuntu 20.04 on AWS.
    
    1. Run the following command: 
       
       ```
       sudo su -
       wget https://raw.githubusercontent.com/GluuFederation/cloud-native-edition/master/automation/startopenabankingdemo.sh && chmod u+x startopenabankingdemo.sh && ./startopenabankingdemo.sh  
       ```
    
    1. [Map](#non-registered-fqdn) your vm ip to the fqdn `demoexample.gluu.org`
      
2. Install Gluu using Helm

    1. Create an Amazon Aurora database with MySQL compatibility version >= `Aurora(MySQL 5.7) 2.07.1` and capacity type `Serverless`. Make sure the EKS cluster can reach the database endpoint. You may choose to use the same VPC as the EKS cluster. Save the master user, master password, and initial database name for use in Gluus helm chart.
    
    1. **Optional if not using istio ingress:** Install [nginx-ingress](https://github.com/kubernetes/ingress-nginx) Helm [Chart](https://github.com/helm/charts/tree/master/stable/nginx-ingress).
    
       ```bash
       kubectl create ns <nginx-namespace>
       helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
       helm repo update
       helm install <nginx-release-name> ingress-nginx/ingress-nginx --namespace=<nginx-namespace>
       ```
    
    1. Copy the [values.yaml](#helm-valuesyaml) below into a file named `override-values.yaml`
    
    1. Modify the values to fit the deployment. This is the time to inject your database connection parameters. You will find the file marked where you need to change the values.
    
    1.  Create a namespace and install:    
       
       ```bash
       kubectl create ns gluu
       helm repo add gluu https://gluufederation.github.io/cloud-native-edition/pygluu/kubernetes/templates/helm
       helm repo update
       helm install <release-name> gluu/gluu -n <namespace> -f override-values.yaml --version=5.0.0
       ```
          
### Non registered FQDN      

If the provided FQDN for Gluu is not globally resolvable map Gluus FQDN at `/etc/hosts` file  to the IP of the lb or microk8s vm as shown below.

  ```bash
  ##
  # Host Database
  #
  # localhost is used to configure the loopback interface
  # when the system is booting.  Do not change this entry.
  ##
  192.168.99.100	demo.openbanking.org # IP and example domain
  127.0.0.1	localhost
  255.255.255.255	broadcasthost
  ::1             localhost
  ```
      
### Uninstalling the Chart

To uninstall/delete `my-release` deployment:

`helm delete <my-release>`

If during installation the release was not defined, release name is checked by running `$ helm ls` then deleted using the previous command and the default release name.

### Enabling mTLS in ingress-nginx

Please note that enabling the following annotations in the values.yaml will enable  [client certificate authentication](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#client-certificate-authentication) on the host level. Uncomment the following from your values.yaml or manually add them to your ingress.

```helmyaml
    additionalAnnotations:
      # Enable client certificate authentication
      nginx.ingress.kubernetes.io/auth-tls-verify-client: "optional"
      # Create the secret containing the trusted ca certificates
      nginx.ingress.kubernetes.io/auth-tls-secret: "default/ca-secret"
      # Specify the verification depth in the client certificates chain
      nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
      # Specify an error page to be redirected to verification errors
      nginx.ingress.kubernetes.io/auth-tls-error-page: "https://demo.openbanking.org/error.html"
      # Specify if certificates are passed to upstream server
      nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
```


### Helm values.yaml

```helmyaml
auth-server:
  dnsPolicy: ""
  dnsConfig: {}
  image:
    pullPolicy: IfNotPresent
    repository: janssenproject/auth-server
    tag: 1.0.0_b1
  replicas: 1
  resources:
    limits:
      cpu: 2500m
      memory: 2500Mi
    requests:
      cpu: 2500m
      memory: 2500Mi
  service:
    authServerServiceName: auth-server
#  livenessProbe:
#    initialDelaySeconds: 25
#  readinessProbe:
#    initialDelaySeconds: 30
  volumes: []
    # Configure any additional volumes that need to be attached to the pod
  volumeMounts: []
    # Configure any additional volumesMounts that need to be attached to the containers

config:
  city: Austin # Change to your city
  configmap:
    cnAuthServerBackend: "auth-server:8080"
    cnSqlDbDialect: mysql
    cnSqlDbHost: gluu.cluster-xxxxxxx.eu-central.rds.amazonaws.com # Change to your Aurora DB endpoint
    cnSqlDbPort: 3306
    cnSqlDbName: jans # Change to your Aurora Database name
    cnSqlDbUser: jans # Change to your Aurora master username
    cnSqlDbTimezone: UTC
    cnSqlPasswordFile: /etc/jans/conf/sql_password
    cnSqldbUserPassword: Test1234# # Change to your Aurora master password
    cnCacheType: NATIVE_PERSISTENCE
    cnConfigKubernetesConfigMap: cn
    # base64 string for the external signing jwks crt and key. Used when .global.cnExtSigningJwksUri is set.
    cnExtSigningJwksCrt: SWFtTm90YVNlcnZpY2VBY2NvdW50Q2hhbmdlTWV0b09uZQo= # Change to your external signing jwks crt
    cnExtSigningJwksKey: SWFtTm90YVNlcnZpY2VBY2NvdW50Q2hhbmdlTWV0b09uZQo= # Change to your external signing jwks key
    cnMaxRamPercent: "75.0"
    cnSecretKubernetesSecret: cn
    containerMetadataName: kubernetes
  countryCode: US # Change to your country
  email: support@gluu.org # Change to your email
  image:
    repository: janssenproject/configuration-manager
    tag: 1.0.0_b1
  orgName: Gluu # Change to your orgnization name
  resources:
    limits:
      cpu: 300m
      memory: 300Mi
    requests:
      cpu: 300m
      memory: 300Mi
  state: TX
  volumes: []
    # Configure any additional volumes that need to be attached to the pod
  volumeMounts: []
    # Configure any additional volumesMounts that need to be attached to the containers
  dnsPolicy: ""
  dnsConfig: {}
config-api:
  dnsPolicy: ""
  dnsConfig: {}
  image:
    pullPolicy: Always
    repository: janssenproject/config-api
    tag: 1.0.0_b1
  replicas: 1
  resources:
    limits:
      cpu: 1000m
      memory: 400Mi
    requests:
      cpu: 1000m
      memory: 400Mi
  service:
    configApiServerServiceName: config-api
#  livenessProbe:
#    initialDelaySeconds: 25
#  readinessProbe:
#    initialDelaySeconds: 30
  volumes: []
    # Configure any additional volumes that need to be attached to the pod
  volumeMounts: []
    # Configure any additional volumesMounts that need to be attached to the containers

global:
  alb:
    ingress: false
  auth-server:
    enabled: true
  awsStorageType: io1
  cnPersistenceType: sql
  cnExtSigningJwksUri: "https://mykeystore.openbanking.wow/xxxxx/xxxxx.jwks" # Change to the external signing jwks uri
  config:
    enabled: true
  #google/kubernetes
  configAdapterName: kubernetes
  #google/kubernetes
  configSecretAdapter: kubernetes
  config-api:
    enabled: true
  fqdn: demo.openbanking.org # Change to your FQDN
  istio:
    enabled: false
    ingress: false
    namespace: istio-system
  nginx-ingress:
    enabled: true
  distribution: openbanking
  persistence:
    enabled: true
  storageClass:
    allowVolumeExpansion: true
    allowedTopologies: []
    mountOptions:
    - debug
    parameters: {}
    #parameters:
      #fsType: ""
      #kind: ""
      #pool: ""
      #storageAccountType: ""
      #type: ""
    provisioner: kubernetes.io/aws-ebs
    reclaimPolicy: Retain
    volumeBindingMode: WaitForFirstConsumer
  upgrade:
    enabled: false
nginx-ingress:
  ingress:
    # /.well-known/openid-configuration
    openidConfigEnabled: true
    # /.well-known/uma2-configuration
    uma2ConfigEnabled: true
    # /.well-known/webfinger
    webfingerEnabled: true
    # /.well-known/simple-web-discovery
    webdiscoveryEnabled: true
    # /jans-config-api
    configApiEnabled: true
    # /.well-known/fido-configuration
    u2fConfigEnabled: true
    # /jans-auth
    authServerEnabled: true
      # in the format of {cert-manager.io/cluster-issuer: nameOfClusterIssuer, kubernetes.io/tls-acme: "true"}
    additionalAnnotations: {}
      # Enable client certificate authentication
      #nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
      # Create the secret containing the trusted ca certificates
      #nginx.ingress.kubernetes.io/auth-tls-secret: "default/ca-secret"
      # Specify the verification depth in the client certificates chain
      #nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
      # Specify an error page to be redirected to verification errors
      #nginx.ingress.kubernetes.io/auth-tls-error-page: "https://demo.openbanking.org/error.html"
      # Specify if certificates are passed to upstream server
      #nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    path: /
    hosts:
    - demo.openbanking.org # Change to your FQDN
    tls:
    - secretName: tls-certificate
      hosts:
      - demo.openbanking.org # Change to your FQDN
persistence:
  dnsPolicy: ""
  dnsConfig: {}
  image:
    pullPolicy: Always
    repository: janssenproject/persistence-loader
    tag: 1.0.0_b1
  resources:
    limits:
      cpu: 300m
      memory: 300Mi
    requests:
      cpu: 300m
      memory: 300Mi
  volumes: []
    # Configure any additional volumes that need to be attached to the pod
  volumeMounts: []
    # Configure any additional volumesMounts that need to be attached to the containers
```