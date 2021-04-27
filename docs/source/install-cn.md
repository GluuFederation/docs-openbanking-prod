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


## Configure cloud or local kubernetes cluster:

=== "EKS"
    ### Amazon Web Services (AWS) - EKS
      
    #### Setup Cluster
    
    -  Follow this [guide](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
     to install a cluster with worker nodes. Please make sure that you have all the `IAM` policies for the AWS user that will be creating the cluster and volumes.
    
    #### Requirements
    
    -   The above guide should also walk you through installing `kubectl` , `aws-iam-authenticator` and `aws cli` on the VM you will be managing your cluster and nodes from. Check to make sure.
    
            aws-iam-authenticator help
            aws-cli
            kubectl version
    
    - **Optional[alpha]:** If using Istio please [install](https://istio.io/latest/docs/setup/install/standalone-operator/) it prior to installing Gluu. You may choose to use any installation method Istio supports. If you have insalled istio ingress , a loadbalancer will have been created. Please save the address of loadblancer for use later during installation.
      
    
=== "Quick start with Microk8s"
    ### MicroK8s
    
    #### Requirements
    
    1.  Create a fresh Ubuntu 20.04 on AWS.
    
    1.  Run the following command: 
       
        ```bash
        sudo su -
        wget https://raw.githubusercontent.com/GluuFederation/cloud-native-edition/master/automation/startopenabankingdemo.sh && chmod u+x startopenabankingdemo.sh && ./startopenabankingdemo.sh  
        ```
    
    1.  [Map](#non-registered-fqdn) your vm ip to the fqdn `demoexample.gluu.org`
      
## Install Gluu using Helm

1.  Create an Amazon Aurora database with MySQL compatibility version >= `Aurora(MySQL 5.7) 2.07.1` and capacity type `Serverless`. Make sure the EKS cluster can reach the database endpoint. You may choose to use the same VPC as the EKS cluster. Save the master user, master password, and initial database name for use in Gluus helm chart.

1.  **Optional if not using istio ingress:** Install [nginx-ingress](https://github.com/kubernetes/ingress-nginx) Helm [Chart](https://github.com/helm/charts/tree/master/stable/nginx-ingress).

    ```bash
    kubectl create ns <nginx-namespace>
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install <nginx-release-name> ingress-nginx/ingress-nginx --namespace=<nginx-namespace>
    ```

1.  Copy the [values.yaml](#helm-valuesyaml) below into a file named `override-values.yaml`

1.  Modify the values to fit the deployment. This is the time to inject your database connection parameters. You will find the file marked where you need to change the values.

1.  Create a namespace and install:    
   
    ```bash
    kubectl create ns gluu
    helm repo add gluu https://gluufederation.github.io/cloud-native-edition/pygluu/kubernetes/templates/helm
    helm repo update
    helm install <release-name> gluu/gluu -n <namespace> -f override-values.yaml --version=5.0.0
    ```
          
## Non registered FQDN      

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
      
## Uninstalling the Chart

To uninstall/delete `my-release` deployment:

`helm delete <my-release>`

If during installation the release was not defined, release name is checked by running `$ helm ls` then deleted using the previous command and the default release name.

## Enabling mTLS in ingress-nginx

1.  Please note that enabling the following annotations in the values.yaml will enable  [client certificate authentication](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#client-certificate-authentication). Uncomment the following from your values.yaml or manually add them to your ingress. 

    ```yaml
        additionalAnnotations:
          # Enable client certificate authentication. Keep this optional. We force it on the path level for /token and /register endpoints.
          nginx.ingress.kubernetes.io/auth-tls-verify-client: "optional"
          # Create the secret containing the trusted ca certificates
          nginx.ingress.kubernetes.io/auth-tls-secret: "gluu/ca-secret"
          # Specify the verification depth in the client certificates chain
          nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
          # Specify if certificates are passed to upstream server
          nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    ```

    In the above secret `ca-secret` inside namespace `gluu` is created.

1.  Set `nginx-ingress.ingress.authServerProtectedToken` and `nginx-ingress.ingress.authServerProtectedRegister` to `true`.
   
### Loading web https certs and keys.

| Associated certificates and keys | Notes                                      |
| -------------------------------- | ------------------------------------------ |
| /etc/certs/web_https.crt         | This is commonly referred to as server.crt |
| /etc/certs/web_https.key         | This is commonly referred to as server.key |
| /etc/certs/web_https.csr         | This is commonly referred to as server.csr |
| /etc/certs/ca.crt                ||
| /etc/certs/ca.key                ||
    

!!! Note
    This will load `web_https.crt`, `web_https.key`, `web_https.csr`, `ca.crt`, and `ca.key` from `/etc/certs`.
    
1.  Create a secret with `web_https.crt`, `web_https.key`, `web_https.csr`, `ca.crt`, and `ca.key`. Note that this may already exist in your deployment.

    ```bash
        kubectl create secret generic web-cert-key --from-file=web_https.crt --from-file=web_https.key --from-file=web_https.csr --from-file=ca.crt --from-file=ca.key -n <gluu-namespace> 
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
              image: janssenproject/certmanager:1.0.0_b2
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

### Example using self signed certs and keys:

1.  Follow the above [steps](#enabling-mtls-in-ingress-nginx) to enable the annotations and protected endpoints before running the [`helm install`](#install-gluu-using-helm) command. 

1.  Wait for services to be in a running state.

1.  Get self signed certs and generate client crt and key. This assumes the namespace Gluu has been installed in is `gluu`.

    ```bash
    mkdir certs
    cd certs
    kubectl get secret cn -o json -n gluu | grep '"ssl_ca_cert":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]' | base64 -d > ca.crt
    kubectl get secret cn -o json -n gluu | grep '"ssl_ca_key":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]' | base64 -d > ca.key
    kubectl get secret cn -o json -n gluu | grep '"ssl_cert":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]' | base64 -d > server.crt
    kubectl get secret cn -o json -n gluu | grep '"ssl_key":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]' | base64 -d > server.key
    openssl req -new -newkey rsa:4096 -keyout client.key -out client.csr -nodes -subj '/CN=Openbanking'
    openssl x509 -req -sha256 -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 02 -out client.crt
    kubectl create secret generic ca-secret -n gluu --from-file=tls.crt=server.crt --from-file=tls.key=server.key --from-file=ca.crt=ca.crt
    cd ..
    ```

1.  Try curling a protected endpoint. 

    1.  Get a client and its associated password. Here, we will use the client id and secret created for config-api.
       
        ```bash
        TESTCLIENT=$(kubectl get cm cn -o json -n gluu | grep '"jca_client_id":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]')
        TESTCLIENTSECRET=$(kubectl get secret cn -o json -n gluu | grep '"jca_client_pw":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]' | base64 -d)
        ```
    
    1.  Curl the protected `/token` endpoint.
    
        ```bash
        curl -X POST -k --cert client.crt --key client.key -u $TESTCLIENT:$TESTCLIENTSECRET https://demo.openbanking.org/jans-auth/restv1/token -d grant_type=client_credentials
        {"access_token":"07688c3e-69ea-403b-a35a-fa3877982c7a","token_type":"bearer","expires_in":299}
        ```
         
    1.  Try using the `jans-cli`:
    
        1.  Clone the [`jans-cli`](https://github.com/JanssenProject/jans-cli/) repo 

            ```
            git clone https://github.com/JanssenProject/jans-cli.git
            ```
            
        1.  Run the jans-cli in interactive mode and try it out: 
           
            ```bash
            ./jans-cli/config-cli.py --host demo.openbanking.org --client-id $TESTCLIENT --client_secret $TESTCLIENTSECRET --cert-file client.crt --key-file client.key -noverify
            ```
           

### Example using self provided certs and keys:

1.  Follow the above [steps](#enabling-mtls-in-ingress-nginx) to enable the annotations and protected endpoints before running the [`helm install`](#install-gluu-using-helm) command. 

1.  Wait for services to be in a running state.

1.  Move all your certs and keys inside one folder called `certs` or any name that is convenient . The necessary certs and keys are highlighted in the [table](#loading-web-https-certs-and-keys). This assumes the namespace Gluu has been installed in is `gluu`.

1.  Follow [instructions](#loading-web-https-certs-and-keys) to rotate all associated certs and keys. This is normally done right after installation. Please note that the fqdn inside the crts and keys must be the same as the one provided during installation.

1.  Move all your certs and keys inside one folder. Generate client side crt, key and create the secret for nginx ingress. This assumes the namespace Gluu has been installed in is `gluu`.

    ```bash
    cd certs
    openssl req -new -newkey rsa:4096 -keyout client.key -out client.csr -nodes -subj '/CN=Openbanking'
    openssl x509 -req -sha256 -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 02 -out client.crt
    kubectl create secret generic ca-secret -n gluu --from-file=tls.crt=web_https.crt --from-file=tls.key=web_https.key --from-file=ca.crt=ca.crt
    cd ..
    ```

1.  By default secret used for TLS `tls-certificate` is created upon installation. This secret must be updated with with the server cert and key `tls.crt=web_https.crt` and `tls.key=web_https.key`.

    ```bash
    kubectl delete secret tls-certificate -n gluu
    kubectl create secret generic tls-certificate --from-file=tls.crt=server.crt --from-file=tls.key=server.key -n gluu
    ```

1.  Try curling a protected endpoint. 

    1.  Get a client and its associated password. Here, we will use the client id and secret created for config-api.
       
        ```bash
        TESTCLIENT=$(kubectl get cm cn -o json -n gluu | grep '"jca_client_id":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]')
        TESTCLIENTSECRET=$(kubectl get secret cn -o json -n gluu | grep '"jca_client_pw":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]' | base64 -d)
        ```
    
    1.  Curl the protected `/token` endpoint.
    
        ```bash
        curl -X POST --cert client.crt --key client.key -u $TESTCLIENT:$TESTCLIENTSECRET https://demo.openbanking.org/jans-auth/restv1/token -d grant_type=client_credentials
        {"access_token":"07688c3e-69ea-403b-a35a-fa3877982c7a","token_type":"bearer","expires_in":299}
        ```

    1.  Try using the `jans-cli`:
    
        1.  Clone the [`jans-cli`](https://github.com/JanssenProject/jans-cli/) repo 

            ```
            git clone https://github.com/JanssenProject/jans-cli.git
            ```
            
        1.  Run the jans-cli in interactive mode and try it out: 
           
            ```bash
            ./jans-cli/config-cli.py --host demo.openbanking.org --client-id $TESTCLIENT --client_secret $TESTCLIENTSECRET --cert-file client.pem --key-file client.key
            ```

## Using jans-cli

`jans-cli` is a Command Line Interface for Gluu Configuration. It also has menu-driven interface that makes it easier to understand how to use Gluu Server through the Interactive Mode.
          
1.  Clone the [`jans-cli`](https://github.com/JanssenProject/jans-cli/) repo 

    ```
    git clone https://github.com/JanssenProject/jans-cli.git
    ```

1.  Get a client and its associated password. Here, we will use the client id and secret created for config-api.
   
    ```bash
    TESTCLIENT=$(kubectl get cm cn -o json -n gluu | grep '"jca_client_id":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]')
    TESTCLIENTSECRET=$(kubectl get secret cn -o json -n gluu | grep '"jca_client_pw":' | sed -e 's#.*:\(\)#\1#' | tr -d '"' | tr -d "," | tr -d '[:space:]' | base64 -d)
    ```
            
1.  Run the jans-cli in interactive mode and try it out: 
   
    ```bash
    ./jans-cli/config-cli.py --host demo.openbanking.org --client-id $TESTCLIENT --client_secret $TESTCLIENTSECRET --cert-file client.pem --key-file client.key
    ```
                 
## Helm values.yaml

```yaml
auth-server:
  dnsPolicy: ""
  dnsConfig: {}
  image:
    pullPolicy: IfNotPresent
    repository: janssenproject/auth-server
    tag: 1.0.0_b2
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
    tag: 1.0.0_b2
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
    tag: 1.0.0_b2
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
    tag: 1.0.0_b2
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