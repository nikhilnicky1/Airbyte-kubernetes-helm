<p align="center">
  <img src="https://github.com/NChittimalla/airbyte-deployment/blob/main/yiH_KxMn_400x400.jpg?raw=true" alt="Terces Logo" />
</p>

# Airbyte Deployment on AWS EKS
### Prerequisites
Before starting the deployment, ensure that the following components are installed:
- Docker
- Kubectl
- Helm
- IAM Credentials
### Technologies Used
- **Airbyte** for data integration
- **AWS Cloud** for infrastructure
- **Kubernetes** for container orchestration
- **RDS PostgreSQL** as the database
- **Amazon S3** for storage
- **Helm Charts** for managing Kubernetes deployments

### Steps for Deployment :
### 1. Configure AWS CLI
Run the following command to configure AWS CLI:
```bash
aws configure
```
#### 2. Set Up AWS Policies
Before configuring the cloud provider, you need to create the necessary AWS IAM Roles and policies. Refer to the [Airbyte AWS policies documentation](https://docs.airbyte.com/deploying-airbyte/infrastructure/aws#policies).

You will need to create an AWS Role and associate that Role with either an AWS User when using Access Credentials, or an Instance Profile or Kubernetes Service Account when using IAM Roles for Service Accounts. That Role will need the following policies depending on it to integrate with S3 and AWS Secret Manager respectively.
#### AWS S3 Policy
The [following policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-policies-s3.html#iam-policy-ex0), allow the cluster to communicate with S3 storage
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::YOUR-S3-BUCKET-NAME"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:PutObjectAcl", "s3:GetObject", "s3:GetObjectAcl", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::YOUR-S3-BUCKET-NAME/*"
    }
  ]
}
```
#### AWS Secrets Manager Policy
The [following policies](https://docs.aws.amazon.com/mediaconnect/latest/ug/iam-policy-examples-asm-secrets.html) allow the cluster to communicate with AWS Secrets Manager:
```json
{
   "Version": "2012-10-17",
   "Statement": [
       {
           "Effect": "Allow",
           "Action": [
               "secretsmanager:GetSecretValue",
               "secretsmanager:CreateSecret",
               "secretsmanager:ListSecrets",
               "secretsmanager:DescribeSecret",
               "secretsmanager:TagResource",
               "secretsmanager:UpdateSecret"
           ],
           "Resource": [
               "*"
           ],
           "Condition": {
               "ForAllValues:StringEquals": {
                   "secretsmanager:ResourceTag/AirbyteManaged": "true"
               }
           }
       }
   ]
}
```
### 3. Installing `eksctl`

`eksctl` is a simple CLI tool for creating and managing Kubernetes clusters on AWS EKS. Follow the steps below for installation on different operating systems.

#### Windows
1. Open Command Prompt or PowerShell as Administrator.
2. Install `eksctl` using Chocolatey:
    ```bash
    choco install eksctl
    ```
3. Verify the installation:
    ```bash
    eksctl version
    ```
#### Ubuntu
1. Download and install `eksctl`:
    ```bash
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin
    ```
2. Verify the installation:
    ```bash
    eksctl version
    ```
#### macOS
1. Install `eksctl` using Homebrew:
    ```bash
    brew tap weaveworks/tap
    brew install weaveworks/tap/eksctl
    ```
2. Verify the installation:
    ```bash
    eksctl version
    ```
---
**Note:** For more information or troubleshooting, refer to the [official `eksctl` documentation](https://eksctl.io/).
### 4. Create an EKS Cluster

- Create an EKS cluster with the desired instance type:
    ```bash
    eksctl create cluster --name airbyte-app --region us-east-2 --nodegroup-name airbyte-nodes --node-type t3.medium --nodes 1 --nodes-min 0 --nodes-max 2 --with-oidc
    ```
    #### What This Command Does
  
    - **`eksctl create cluster`**: Initializes the creation of a new Amazon EKS (Elastic Kubernetes Service) cluster.
    - **`--name airbyte-app`**: Sets the name of the EKS cluster to `airbyte-app`.
    - **`--region us-east-1`**: Specifies the AWS region where the EKS cluster will be created (`us-east-1` in this case).
    - **`--nodegroup-name airbyte-nodes`**: Names the node group in the EKS cluster as `airbyte-nodes`.
    - **`--node-type t3.medium`**: Defines the instance type for the nodes in the node group as `t3.medium`.
    - **`--nodes 1`**: Configures the cluster to start with 1 node in the node group.
    - **`--nodes-min 0`**: Sets the minimum number of nodes in the node group to 0, allowing the cluster to scale down to zero nodes if needed.
    - **`--nodes-max 2`**: Sets the maximum number of nodes in the node group to 2.
    - **`--with-oidc`**: Enables the OpenID Connect (OIDC) provider integration for the EKS cluster, which is often needed for certain Kubernetes workloads and IAM roles.
    This command sets up an EKS cluster with a node group of `t3.medium` instances, starting with 2 nodes and allowing for autoscaling within the defined range. The OIDC provider integration is also     
    enabled, which is useful for Kubernetes service accounts and IAM role management.

### 5. Create RDS PostgreSQL Database
- For production deployments, we recommend using a dedicated database instance for better reliability, and backups (such as AWS RDS or GCP Cloud SQL) instead of the default internal Postgres database (airbyte/db) that Airbyte spins up within the Kubernetes cluster, follow the link  below to create RDS PostgresSQL Database.
  
    - [RDS PostgreSQL instance in AWS](https://docs.google.com/document/d/1uaQ4dlR3Fl6GC8J7c-AgpgiZBfBwwUQvqMgriF2TV_0/edit#heading=h.nmdla6v4cowy)
      
⚠️ **Important Note:** Ensure that the Airbyte and RDS instances are deployed within the same VPC and are associated with the same Security Group, with inbound rules configured to allow connections on port 5432 and ALL TCP traffic.

### 6. Create Amazon S3 Bucket
- Airbyte recommends using an object storage solution such as S3 and GCS for storing State and Logging information. You must select which type of blob store that you wish to use. Currently, S3 and GCS are supported. If you are using an S3 compatible solution, use the S3 type and provide an endpoint key/value as needed
  
    - [Amazon S3 Bucket in AWS](https://docs.google.com/document/d/1Fdc_yo3er9rcBwp61uWlZ91Dewsjs85QOJkbyDN5_Oo/edit#heading=h.u5izzjwtpj51)

### 7.  Configure Kubectl
- Open the Terminal & Run the following commands to configure Kubectl for your EKS cluster:
    ```bash
    eksctl utils write-kubeconfig --cluster=airbyte-app --region us-east-2
    kubectl config get-contexts
    kubectl config use-context <eks context>
    ```
### 8. Create Namespace
- Create a namespace for Airbyte:
    ```bash
    kubectl create namespace airbyte
    ```
### 9. Configure Secrets for AWS Secret Manager and PostgreSQL
- Create secret.yaml and configure all the secrets in one file
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: airbyte-config-secrets
  type: Opaque
  stringData:
    # AWS Secret Manager
    AWS_ACCESS_KEY_ID: xxxxxxxxxxxxxx
    AWS_SECRET_ACCESS_KEY: xxxxxxxxxxxxxx
  
    # Database Secrets
    DATABASE_PASSWORD: xxxxxxxxxxxxx
  ```
### 10. Apply the configuration for secrets:
- Create the file by using the kubectl command
    ```bash
    kubectl apply -f secret.yaml -n airbyte
    ```
### 11. Create a value.yaml override file
To configure your installation of Airbyte, you will need to override specific parts of the Helm Chart. To do this you should create a new file called values.yaml somewhere that is accessible during the installation process.
```yaml
global:
  state:
    storage:
      type: S3
  database:
    type: external
    host: xxxxxx # hostname
    port: 5432
    database: postgres # By default Database name is postgres orelse give the name which you have created
    user: airbyte
    secretName: airbyte-config-secrets # Ensure this matches the secret name
    passwordSecretKey: DATABASE_PASSWORD
  logs:
    accessKey:
      existingSecret: airbyte-config-secrets # Ensure this matches the secret name
      existingSecretKey: AWS_ACCESS_KEY_ID
    secretKey:
      existingSecret: airbyte-config-secrets
      existingSecretKey: AWS_SECRET_ACCESS_KEY
  storage:
    type: S3
    bucket:
      activityPayload: xxxxxxxxxx # s3 bucket name
      log: xxxxxxxxxx # s3 bucket name
      state: xxxxxxxxxx # s3 bucket name
      workloadOutput: xxxxxxxxxx # s3 bucket name
    s3:
      region: "us-east-2"
      authenticationType: "credentials"
      accessKeyIdSecretKey: "AWS_ACCESS_KEY_ID"
      secretAccessKeySecretKey: "AWS_SECRET_ACCESS_KEY"
  minio:
    enabled: false

  secretsManager:
    type: "awsSecretManager"
    secretsManagerSecretName: "airbyte-config-secrets" # Ensure this matches the secret name
    awsSecretManager:
      region: "us-east-2"
      authenticationType: "credentials"
      accessKeyIdSecretKey: "AWS_ACCESS_KEY_ID"
      secretAccessKeySecretKey: "AWS_SECRET_ACCESS_KEY"

worker:
  env:
    - name: EXTERNAL_URL
      value: "https://airbyte.wowparlour.shop" # Provide your own domain here
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets" # Ensure this matches the secret name
          key: "DATABASE_PASSWORD"
    - name: S3_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets" # Ensure this matches the secret name
          key: "AWS_ACCESS_KEY_ID"
    - name: S3_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets" # Ensure this matches the secret name
          key: "AWS_SECRET_ACCESS_KEY"
    - name: AWS_SECRET_MANAGER_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets" # Ensure this matches the secret name
          key: "AWS_ACCESS_KEY_ID"
    - name: AWS_SECRET_MANAGER_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets" # Ensure this matches the secret name
          key: "AWS_SECRET_ACCESS_KEY"

server:
  env:
    - name: EXTERNAL_URL
      value: "https://airbyte.wowparlour.shop" # Provide your own domain here
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "DATABASE_PASSWORD"
    - name: S3_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "AWS_ACCESS_KEY_ID"
    - name: S3_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "AWS_SECRET_ACCESS_KEY"
    - name: AWS_SECRET_MANAGER_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "AWS_ACCESS_KEY_ID"
    - name: AWS_SECRET_MANAGER_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "AWS_SECRET_ACCESS_KEY"

workloadLauncher:
  env:
    - name: EXTERNAL_URL
      value: "https://airbyte.wowparlour.shop" # Provide your own domain here
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "DATABASE_PASSWORD"
    - name: S3_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "AWS_ACCESS_KEY_ID"
    - name: S3_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "AWS_SECRET_ACCESS_KEY"
    - name: AWS_SECRET_MANAGER_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "AWS_ACCESS_KEY_ID"
    - name: AWS_SECRET_MANAGER_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: "airbyte-config-secrets"
          key: "AWS_SECRET_ACCESS_KEY"

postgresql:
  enabled: false

externalDatabase:
  host: xxxxxxx # hotsname
  port: 5432
  database: airbyte_db
  user: airbyte
  existingSecret: airbyte-config-secrets
  existingSecretPasswordKey: DATABASE_PASSWORD
  jdbcUrl: jdbc:postgresql://production-ts-airbytexxxxxxxxxxxx.us-east-2.rds.amazonaws.com:5432/postgres?ssl=true&sslmode=require
```
### 12. Automate SSL Certificate Management with Cert-Manager
  For automated certificate issuance and renewal, consider using Cert-Manager with Let's Encrypt.
  
  ### a. Install Cert-Manager:
  ```bash
  kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
  ```
  ### b. configure a ClusterIssuer:
  Create a ClusterIssuer for Let's Encrypt-(cluster-issuer.yaml):
  ```yaml
  apiVersion: cert-manager.io/v1
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-prod
  spec:
    acme:
      server: https://acme-v02.api.letsencrypt.org/directory
      email: <your-email>
      privateKeySecretRef:
        name: letsencrypt-prod-private-key
      solvers:
        - http01:
            ingress:
              class: nginx
  ```
  #### Apply the Configuration:
  ```bash
  kubectl apply -f cluster-issuer.yaml -n airbyte
  ```
  ### c. create an airbyte-tls.yaml file
  ```yaml
  apiVersion: cert-manager.io/v1
  kind: Certificate
  metadata:
    name: airbyte-tls
    namespace: airbyte
  spec:
    secretName: airbyte-tls
    issuerRef:
      group: cert-manager.io
      kind: ClusterIssuer
      name: letsencrypt-prod
    commonName: airbyte.wowparlour.shop # enter your own domain
    dnsNames:
      - airbyte.wowparlour.shop # enter your own domain
  ```
  #### Apply the Configuration:
  ```bash
  kubectl apply -f airbyte-tls.yaml -n airbyte
  ```
  ### d. update Ingress Annotations:
  Ensure your Ingress resource includes annotations for Cert-Manager to manage the certificates.

we can configure [Ingress](https://docs.airbyte.com/deploying-airbyte/integrations/ingress) by following the directions for your specific Ingress provider.

To access the Airbyte UI, we will need to manually attach an ingress configuration to your deployment. The following is a simplified definition of an ingress resource you could use for your Airbyte instance: Nginx

If you don't already have an NGINX controller installed, you can do it by running below command or following the instructions from NGINX.
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx -n airbyte --set controller.publishService.enabled=true
```
### 13. Create an ingress.yaml file
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: airbyte-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  labels:
    kubernetes.io/ingress.class: "nginx"
spec:
  ingressClassName: nginx
  rules:
    - host: <your-own-domain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: airbyte-airbyte-webapp-svc
                port:
                  number: 80
  tls:
    - hosts:
        - <your-own-domain>
      secretName: airbyte-tls
```
#### Apply the Configuration:
```bash
kubectl apply -f ingress.yaml -n airbyte
```
You can now access the UI in your browser at: https://your-own-domain

### 14. DNS Configuration

```bash
kubectl get svc -n airbyte
```
You will see a list of all the services deployed in your namespace, including the ingress service, which will display the External IP address.

### 15. Steps to Update DNS Records

- Log in to Your DNS Provider's Dashboard:
- Go to the DNS management section for your own domain (example.com).
- Add or Update DNS Records:

#### For A Record:

- Set the record type to A.
- Set the name to airbyte (or @ if it’s for the root domain).
- Set the value to 10.xx.x.20 # use your own ip address
- Save the changes.

#### For CNAME Record:

- Set the record type to CNAME.
- Set the name to airbyte (or @ if it’s for the root domain).
- Set the value to a34706dc6e8464d5689edxxxxxxxxx902.us-east-2.elb.amazonaws.com # use your own external ip
- Save the changes.

Verify DNS Propagation:

Use tools like DNS Checker to check if the DNS records have been updated and are propagating correctly.

### 16. Add the Helm Repository​
The deployment will use a Helm chart which is a package for Kubernetes applications, acting like a blueprint or template that defines the resources needed to deploy an application on a Kubernetes cluster. Charts are stored in helm-repo.
- To add a remote helm repo:
    ```bash
    helm repo add airbyte https://airbytehq.github.io/helm-charts
    ```
- After adding the repo, perform the repo indexing process by running 
    ```bash
    helm repo update
    ```
- You can now browse all charts uploaded to repository by running 
    ```bash
    helm search repo airbyte
    ```
### 17. Installing Airbyte​

After you have applied your Secret values to the Cluster and you have filled out a values.yaml file appropriately for your specific configuration, you can begin a Helm Install. To do this, make sure that you have the [Helm Client](https://helm.sh/docs/intro/install/) installed and on your path. Then you can run:
```bash
helm install airbyte airbyte/airbyte --namespace airbyte --values <path-to-values.yaml-file>/values.yaml --timeout 120m
```
### 18. To destroy the cluster use the below command : 
```bash
eksctl delete cluster --name airbyte-app
```
⚠️ **Important Note:** Ensure that the Airbyte and RDS instances are deployed within the same VPC and are associated with the same Security Group, with inbound rules configured to allow connections on port 5432 and ALL TCP traffic.

### 19. Reference Documents : 
1. [Deploying Airbyte](https://docs.airbyte.com/deploying-airbyte/)
2. [Storage Integrations](https://docs.airbyte.com/deploying-airbyte/integrations/storage)
3. [Secrets Integrations](https://docs.airbyte.com/deploying-airbyte/integrations/secrets)
4. [Database Integrations](https://docs.airbyte.com/deploying-airbyte/integrations/database)
5. [Ingress Integrations](https://docs.airbyte.com/deploying-airbyte/integrations/ingress)
