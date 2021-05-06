# Gluu Openbanking Cloud Native Production tutorial setup

## Cloud Native Distribution


## Getting Started with Kubernetes


## System Requirements for cloud deployments

Please calculate the minimum required resources as per services deployed. The following table contains default recommended resources to start with. Depending on the use of each service the resources may be increased or decreased. 

|Service           | CPU Unit   |    RAM      |   Disk Space     | Processor Type | Required                                    |
|------------------|------------|-------------|------------------|----------------|---------------------------------------------|
|Auth-server            | 2.5        |    2.5GB    |   N/A            |  64 Bit        | Yes                                         |
|config - job      | 0.5        |    0.5GB    |   N/A            |  64 Bit        | Yes on fresh installs                       |
|persistence - job | 0.5        |    0.5GB    |   N/A            |  64 Bit        | Yes on fresh installs                       |
|nginx             | 1          |    1GB      |   N/A            |  64 Bit        | Yes.                                        |


## Configure EKS with Aurora

### Amazon Web Services (AWS) - EKS
  
#### Setup Cluster

-  Follow this [guide](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
 to install a cluster with worker nodes. This setup must used three `t2.xlarge` instances distributed in three different zones. Please make sure that you have all the `IAM` policies for the AWS user that will be creating the cluster and volumes.

#### Requirements

-   The above guide should also walk you through installing `kubectl` , `aws-iam-authenticator` and `aws cli` on the VM you will be managing your cluster and nodes from. Check to make sure.

        aws-iam-authenticator help
        aws-cli
        kubectl version
  
### Amazon Aurora

[Amazon Aurora](https://aws.amazon.com/rds/aurora/?aurora-whats-new.sort-by=item.additionalFields.postDateTime&aurora-whats-new.sort-order=desc) is a MySQL and PostgreSQL-compatible relational database built for the cloud, that combines the performance and availability of traditional enterprise databases with the simplicity and cost-effectiveness of open source databases. Gluu fully supports Amazon Aurora, and recommends it in production.

1.  Copy the [values.yaml](#helm-valuesyaml) below into a file named `override-values.yaml`. We will be referencing this file throughout this tutorial.
 
1.  Create an Amazon Aurora database with MySQL compatibility version >= `Aurora(MySQL 5.7) 2.07.1` and capacity type `Serverless`. Make sure the EKS cluster can reach the database endpoint. You may choose to use the same VPC as the EKS cluster. Save the master user, master password, and initial database name for use in Gluus helm chart.

1.  Inject the Aurora endpoint, master user, master password, and initial database name for use in Gluus helm chart [`override-values.yaml`](#helm-valuesyaml). 

    |Helm values configuration                | Description                                                                     | default      |
    |-----------------------------------------|---------------------------------------------------------------------------------|--------------|
    |config.configmap.cnSqlDbHost             | Aurora database endpoint i.e `gluu.cluster-xxxxxxx.eu-central.rds.amazonaws.com`|    empty     |
    |config.configmap.cnSqlDbPort             | Aurora database port                                                            |    `3306`    |
    |config.configmap.cnSqlDbName             | Aurora initial database name                                                    |    `jans`    |
    |config.configmap.cnSqlDbUser             | Aurora master user                                                              |    `jans`    |
    |config.configmap.cnSqldbUserPassword     | Aurora master password                                                          |   `Test1234#`|


## Install Gluu using Helm

1.  Install [nginx-ingress](https://github.com/kubernetes/ingress-nginx) Helm [Chart](https://github.com/helm/charts/tree/master/stable/nginx-ingress).

    ```bash
    kubectl create ns <nginx-namespace>
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    # The below `set`  is needed currently as the admission webhook of nginx upon initial installation sometimes revokes the ingress definitions incorrectly.
    helm install <nginx-release-name> ingress-nginx/ingress-nginx --namespace=<nginx-namespace> --set controller.admissionWebhooks.enabled=false
    ```
    
1.  Create a namespace  for gluu openbanking distribution:    
   
    ```bash
    kubectl create ns gluu
    ```
    
1.  Enable mTLS. 

    | certificates and keys of interest in mTLS | Notes                                      |
    | ----------------------------------------  | ------------------------------------------ |
    | web_https.crt         | This is commonly referred to as server.crt |
    | web_https.key         | This is commonly referred to as server.key |
    | web_https.csr         | This is commonly referred to as server.csr |
    | ca.crt                ||
    | ca.key                ||
    
    These are used by nginx ingress for client-authentication and TLS. These may be provided to you by your domain provider or may be [self-signed](https://kubernetes.github.io/ingress-nginx/examples/PREREQUISITES/#client-certificate-authentication) and maintained.
     
    1.  Please note that enabling the following annotations in the values.yaml will enable  [client certificate authentication](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#client-certificate-authentication). Uncomment the following from the helm charts [`override-values.yaml`](#helm-valuesyaml)

        ```yaml
            additionalAnnotations:
              # Enable client certificate authentication. Keep this optional. We force it on the path level for /token and /register endpoints.
              nginx.ingress.kubernetes.io/auth-tls-verify-client: "optional"
              # Create the secret containing the trusted ca certificates
              nginx.ingress.kubernetes.io/auth-tls-secret: "gluu/tls-ca-certificate"
              # Specify the verification depth in the client certificates chain
              nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
              # Specify if certificates are passed to upstream server
              nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
        ```
        
    1.  Set `nginx-ingress.ingress.authServerProtectedToken` and `nginx-ingress.ingress.authServerProtectedRegister` in the helm charts [`override-values.yaml`](#helm-valuesyaml) to `true`.
   
    1.  Create a secret containing the CA certificate and the Server Certificate which is Signed by the CA. For more information read [here](https://kubernetes.github.io/ingress-nginx/examples/auth/client-certs/).
    
        ```bash
        kubectl create secret generic tls-ca-certificate -n gluu --from-file=tls.crt=web_https.crt --from-file=tls.key=web_https.key --from-file=ca.crt=ca.crt
        ```
        
1.  Inject OBIE signed certs, keys and uri: 

    1.  When using OBIE signed certificates and keys, there are three main objects to have. The certificate pem file i.e `obsigning.pem`, the key i.e `obsigning-oajsdij8927123.key`, and the jwks uri `https://mykeystore.openbanking.wow/xxxxx/xxxxx.jwks`.
    
    1.  base64 encrypt both the `.pem` and `.key` as they will be injected as base64 strings inside the helm [`override-values.yaml`](#helm-valuesyaml).
    
        ```bash
        cat obsigning.pem | base64 | tr -d '\n' > obsigningbase64.pem
        cat obsigning-oajsdij8927123.key | base64 | tr -d '\n' > obsigningbase64.key
        ```
        
    1.  Copy the base64 string in `obsigningbase64.pem` into the helm chart [`override-values.yaml`](#helm-valuesyaml) at `config.configmap.cnExtSigningJwksCrt`
    
    1.  Copy the base64 string in `obsigningbase64.key` into the helm chart [`override-values.yaml`](#helm-valuesyaml)  at `config.configmap.cnExtSigningJwksKey`
    
    1.  Add the jwks uri to the helm chart [`override-values.yaml`](#helm-valuesyaml) at `global.cnExtSigningJwksUri`
    
    |Helm values configuration                | Description                                                                                         | default      |
    |-----------------------------------------|-----------------------------------------------------------------------------------------------------|--------------|
    |global.cnExtSigningJwksUri               | external signing jwks uri string                                                                    |    empty     |
    |config.configmap.cnExtSigningJwksCrt     | base64 string for the external signing jwks crt. Activated when .global.cnExtSigningJwksUri is set  |    empty     |
    |config.configmap.cnExtSigningJwksKey     | base64 string for the external signing jwks key . Activated when .global.cnExtSigningJwksUri is set |    empty     |

1.  Modify other values in helm chart [`override-values.yaml`](#helm-valuesyaml) configuration to fit your setup

    |Helm values configuration                | Description                                                                                         | default      |
    |-----------------------------------------|--------------------------------|-----------------------------|
    |config.city                              | City                           |    `Austin`                 |
    |config.countryCode                       | Country                        |    `US`                     |
    |config.email                             | Email                          |    `support@gluu.org`       |
    |config.orgName                           | Organization name              |    `Gluu`                   |
    |config.state                             | State                          |    `TX`                     |
    |global.fqdn                              | FQDN                           |    `demo.openbanking.org`   |
    |nginx-ingress.ingress.hosts              | A list containing the FQDN     |    `[demo.openbanking.org]` |
    |nginx-ingress.ingress.tls.hosts          | A list containing the FQDN     |    `[demo.openbanking.org]` |

    Please note that the FQDN **must** be resolvable or the `config-api` will fail to find the host. You may choose to modify the `CoreDNS` to reroute the domain internally to the nginx-ingress as following:
    
    Add rewrite rule to CoreDNS `ConfigMap`. In the following example we will be using ingress-nginx which you have installed previously. The internal address of this service will be `nginx-ingress-nginx-controller.nginx.svc.cluster.local` which is in the following format `<releasename>-ingress-nginx-controller.<namespace>.svc.cluster.local` and our example domain will be `demo.openbanking.org`:
    
    1.  Take a copy of the `ConfigMap`:
    
        ```bash
        kubectl -n kube-system get configmap coredns -o yaml > original_coredns_cm.yaml
        cp original_coredns_cm.yaml gluu_modified_coredns_cm.yaml 
        ```
    
    1.  Edit the `ConfigMap` to include the rewrite using any editior:
    
        ```bash
        vi gluu_modified_coredns_cm.yaml
        ```
       
        ```yaml
           .:53 {
                    ....
                    rewrite name demo.openbanking.org nginx-ingress-nginx-controller.nginx.svc.cluster.local
                }
        ```
    
    1.  Apply the changes:
    
        ```bash
        kubectl apply -f gluu_modified_coredns_cm.yaml
        ```    
    
1.  Install gluu openbanking distribution:    
   
    ```bash
    helm repo add gluu https://gluufederation.github.io/cloud-native-edition/pygluu/kubernetes/templates/helm
    helm repo update
    helm install <release-name> gluu/gluu -n <namespace> -f override-values.yaml --version=5.0.0
    ```
  
1.  Wait for your pods to be up and running.  

1.  Load your web https certs and keys.

    | Associated certificates and keys | Notes                                      |
    | -------------------------------- | ------------------------------------------ |
    | /etc/certs/web_https.crt         | This is commonly referred to as server.crt |
    | /etc/certs/web_https.key         | This is commonly referred to as server.key |
    | /etc/certs/web_https.csr         | This is commonly referred to as server.csr |
    | /etc/certs/ca.crt                ||
    | /etc/certs/ca.key                ||
        
    
    !!! Note
        This will load `web_https.crt`, `web_https.key`, `web_https.csr`, `ca.crt`, and `ca.key` to `/etc/certs`. This step is important in order for mTLS to fully work as nginx-ingress will pass the client certificate down to the auth-server and the auth-server will validate the client certificate.
        
    1.  Create a secret with `web_https.crt`, `web_https.key`, `web_https.csr`, `ca.crt`, and `ca.key`. Note that this may already exist in your deployment.
    
        ```bash
            kubectl create secret generic web-cert-key --from-file=web_https.crt --from-file=web_https.key --from-file=web_https.csr --from-file=ca.crt --from-file=ca.key -n <gluu-namespace>
        ```
               
        If using the common server names:
        
        ```bash
        kubectl create secret generic web-cert-key --from-file=web_https.crt=server.crt --from-file=web_https.key=server.key --from-file=web_https.csr=server.csr --from-file=ca.crt --from-file=ca.key -n <gluu-namespace>
        ```
        
        
    1.  Create a file named `load-web-key-rotation.yaml` with the following contents :
                       
        ```yaml
        # License terms and conditions for Gluu Cloud Native Edition:
        # https://www.apache.org/licenses/LICENSE-2.0
        apiVersion: batch/v1
        kind: Job
        metadata:
          name: load-web-key-rotation
        spec:
          template:
            metadata:
              annotations:
                sidecar.istio.io/inject: "false"                  
            spec:
              restartPolicy: Never
              volumes:
              - name: web-cert
                secret:
                  secretName: web-cert-key
                  items:
                    - key: web_https.crt
                      path: web_https.crt
              - name: web-key
                secret:
                  secretName: web-cert-key
                  items:
                    - key: web_https.key
                      path: web_https.key
              - name: web-csr
                secret:
                  secretName: web-cert-key
                  items:
                    - key: web_https.csr
                      path: web_https.csr
              - name: web-ca-cert
                secret:
                  secretName: web-cert-key
                  items:
                    - key: ca.crt
                      path: ca.crt
              - name: web-ca-key
                secret:
                  secretName: web-cert-key
                  items:
                    - key: ca.key
                      path: ca.key                              
              containers:
                - name: load-web-key-rotation
                  image: janssenproject/certmanager:1.0.0_b3
                  envFrom:
                  - configMapRef:
                      name: gluu-config-cm  #This may be different in your Helm setup
                  volumeMounts:
                    - name: web-cert
                      mountPath: /etc/certs/web_https.crt
                      subPath: web_https.crt
                    - name: web-key
                      mountPath: /etc/certs/web_https.key
                      subPath: web_https.key
                    - name: web-csr
                      mountPath: /etc/certs/web_https.csr
                      subPath: web_https.csr
                    - name: web-ca-cert
                      mountPath: /etc/certs/ca.crt
                      subPath: ca.crt
                    - name: web-ca-key
                      mountPath: /etc/certs/ca.key
                      subPath: ca.key   
                  args: ["patch", "web", "--opts", "source:from-files"]
        ```
        
    1.  Apply job
    
        ```bash
            kubectl apply -f load-web-key-rotation.yaml -n <gluu-namespace>
        ```
          
1.  Preform a rolling restart for the auth-server and config-api

    ```bash
    kubectl rollout restart deployment <gluu-relase-name>-auth-server -n <gluu-namespace>
    kubectl rollout restart deployment <gluu-relase-name>-config-api -n <gluu-namespace>
    #kubectl rollout restart deployment gluu-auth-server -n gluu
    ```
          
## Testing the setup

`jans-cli` is a Command Line Interface for Gluu Configuration. It also has menu-driven interface that makes it easier to understand how to use Gluu Server through the Interactive Mode.
          
1. Download [`jans-cli.pyz`](https://github.com/JanssenProject/jans-cli/releases). This package can be built [manually](https://github.com/JanssenProject/jans-cli#build-jans-clipyz-manually).

1.  Get a client and its associated password. Here, we will use the client id and secret created for config-api.
   
    ```bash
    TESTCLIENT=$(kubectl get cm cn -o json -n gluu | grep '"jca_client_id":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]')
    TESTCLIENTSECRET=$(kubectl get secret cn -o json -n gluu | grep '"jca_client_pw":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]' | base64 -d)
    ```
            
1.  Using your `ca.crt` and `ca.key` that was provided during setup generate as many client certificates and keys as needed.

    ```bash
    openssl req -new -newkey rsa:4096 -keyout client.key -out client.csr -nodes -subj '/CN=My Client'
    openssl x509 -req -sha256 -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 02 -out client.crt
    ```
            
1.  Run the jans-cli in interactive mode and try it out: 
   
    ```bash
    python3 jans-cli-linux-amd64.pyz --host demo.openbanking.org --client-id $TESTCLIENT --client_secret $TESTCLIENTSECRET --cert-file client.crt --key-file client.key
    ```
    
Go have fun and test more [scenarios](https://gluu.org/docs/openbanking/configuration-instructions/).

## Cleaning up

1.  Delete your Amazon Aurora database

1.  Uninstall Gluu helm chart

    ```bash
    helm delete <my-release> -n <gluu-namespace>
    ```
    
1.  Delete Gluus namespace

    ```bash
    kubectl delete -n <gluu-namespace>
    ```
 
1.  Delete your EKS cluster

    ```bash
    eksctl delete cluster <gluu-eks-cluster>
    ```    

                 
## Helm values.yaml

```yaml
auth-server:
  dnsPolicy: ""
  dnsConfig: {}
  image:
    pullPolicy: IfNotPresent
    repository: janssenproject/auth-server
    tag: 1.0.0_b3
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
    #- name: jose4j
    #  configMap:
    #    name: jose4j
  volumeMounts: []
    # Configure any additional volumesMounts that need to be attached to the containers
    #- name: jose4j
    #  mountPath: "/opt/jans/jetty/jans-auth/custom/libs/jose4j-0.7.7.jar"
    #  subPath: jose4j-0.7.7.jar
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
    tag: 1.0.0_b3
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
    tag: 1.0.0_b3
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
    #/jans-auth/restv1/token
    authServerProtectedToken: false
    #/jans-auth/restv1/register
    authServerProtectedRegister: false
      # in the format of {cert-manager.io/cluster-issuer: nameOfClusterIssuer, kubernetes.io/tls-acme: "true"}
    additionalAnnotations: {}
      # Enable client certificate authentication
      #nginx.ingress.kubernetes.io/auth-tls-verify-client: "optional"
      # Create the secret containing the trusted ca certificates
      #nginx.ingress.kubernetes.io/auth-tls-secret: "gluu/tls-ca-certificate"
      # Specify the verification depth in the client certificates chain
      #nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
      # Specify if certificates are passed to upstream server
      #nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    path: /
    hosts:
    - demo.openbanking.org # Change to your FQDN
    tls:
    - secretName: tls-ca-certificate
      hosts:
      - demo.openbanking.org # Change to your FQDN
persistence:
  dnsPolicy: ""
  dnsConfig: {}
  image:
    pullPolicy: Always
    repository: janssenproject/persistence-loader
    tag: 1.0.0_b3
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