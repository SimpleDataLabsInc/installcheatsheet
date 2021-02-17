This document lists down commands to bring up Prophecy setup quickly. We highly recommend to go over our architecture and detailed install instructions on our [install](https://docs.prophecy.io/deployment/install/) page.

### Setup ProService(ucp) using Helm
#### Deploy TLS secrets for ingress
We shall be using DNS Zones hosted in Prophecy landscape for setup. `ucp-tls-secret` is passed to ingress for TLS conn.
* Create TLS secrets
```shell
openssl req -x509 -sha256 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 356 -nodes -subj ‘/CN=My Cert Authority’
openssl req -new -newkey rsa:4096 -keyout server.key -out server.csr -nodes -subj ‘/CN=*.cloud.prophecy.io’
openssl x509 -req -sha256 -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt
```
* Deploy the secrets in `ucp` namespace
```shell
kubectl create namespace ucp
kubectl create secret generic ucp-tls-secret --from-file=tls.crt=server.crt --from-file=tls.key=server.key -n ucp
```

#### Setup Postgres instance with necessary database schemas
* Run the `SQL` queries to create the user and databases needed for prophecy. User `sdl` expects a specific password (*will share seperately*).
```sql
CREATE DATABASE prophecy;
CREATE USER sdl WITH PASSWORD '******'; #will share seperately
CREATE DATABASE exec;
CREATE DATABASE gogs;
CREATE DATABASE asp;
CREATE DATABASE superset;
CREATE DATABASE airflow;
GRANT ALL PRIVILEGES ON DATABASE asp TO sdl;
GRANT ALL PRIVILEGES ON DATABASE exec TO sdl;
GRANT ALL PRIVILEGES ON DATABASE airflow TO sdl;
GRANT ALL PRIVILEGES ON DATABASE gogs TO sdl;
GRANT ALL PRIVILEGES ON DATABASE superset TO sdl;
GRANT ALL PRIVILEGES ON DATABASE airflow TO sdl;
```

#### Create a values.yaml file
We need to create a `values.yaml` file to pass to helm during to pass during `helm install`.
```yaml
ucp:
  hostname: <ingress FQDN edit here> #This should be added to the DNS zone to point to the loadbalancer endpoint.
  certSecretName: ucp-tls-secret

env:
  awsAccessKeyID: <edit here>
  awsSecretAccessKey: <edit here>
  awsDefaultRegion: <edit here>

postgres:
  server: <edit here. URL of the Postgres server instance>
  port: <edit here>
  user: <edit here>
  secret:
    passwdValue: <password for the Postgres server instance>
```

#### Install `ucp` helm chart
```shell
helm repo add prophecy http://simpledatalabsinc.github.io/prophecy/
helm install ucp prophecy/ucp -f values.yaml -n ucp --version 0.0.1
```

#### Find the ingress associated with ucp and add the domain to the DNS zone
```shell
kubectl get ingress -n ucp
NAME   HOSTS                       ADDRESS                                                                  PORTS     AGE
ucp    foo.cloud.prophecy.io   abcdefghighe-123456789.us-west-1.elb.amazonaws.com   80, 443   7m20s
```
For the example above, add a CNAME record for domain `foo.cloud.prophecy.io` pointing to `abcdefghighe-123456789.us-west-1.elb.amazonaws.com`.

### Deploy ProCtl

Download prophecy management CLI (ProCtl) and connect with ProService
   - proctl supports mac/linux environment and connects to a ProService with a single command
   ```
   proctl -s <ProService IP Address> -p 443 -k
   ```

*Contact Prophecy team for the links to the latest versions of Proctl and helm chart.*


### Customer 
1. Create/Onboard a new customer in ProService using below command in proctl:
```
proctl » customer create -n <customer-name> --email <customer-email>
```
A sample command to onboard a customer say _'abc'_ would look like:
```
proctl » customer create -n abc --email abc@gmail.com
```

2. The below command sets the context to the given customer and thereafter all operations are done in context of that customer.
```
proctl » context set -c <customer-name>
```
A sample command to set context to customer _'abc'_ along with sample response would look like:
```
proctl » context set -c abc
```
Response:
```
Context has been set to customer abc
proctl [abc] »
```
As shown above, the shell prompt changes to `proctl [abc]` from `proctl`

### K8s cluster 

1. Please run below command to attach an existing k8s cluster with ProService. This command adds an existing kubernetes cluster to ProService and makes it available for rest of the deployment. 


```
proctl [<customer>] » cluster add aws -n <cluster-name> --aws-access-key-id=<access-key-id> --aws-secret-access-key=<secret-access-key> --aws-efs-id=<efs-id> --aws-efs-name=<efs-name> --aws-region=<region> --kubeconfig <absolute path to kubeconfig file of cluster>
```
Please use below commands to check the status of cluster:
```
AWS:
proctl [<customer>] » cluster get aws -n <cluster-name> 
```

*Cluster add takes less than a minute to complete.*
*kubeconfig takes the absolute path to kubeconfig file on machine where proctl is running.*


### Prophecy Platform
Prophecy Platform is responsible for backup,restore,logs, metrics and auto-scaling for prophecy setup. Please run below command to create prophecy platform on a given k8s cluster.  

```
proctl [<customer>] » platform create -n <platform-name> --cluster <cluster-name> --version 0.0.2
```
Creating a platform is a long operation and one can track the status of operation with 'platform get' command. Please use below command to check the status of platform creation:

```
proctl [<customer>] » platform get --cluster <cluster-name> 
```
*Platform creation takes around 5 minutes to complete.*


### Control Plane
Control Plane represents one installation of Prophecy Application. Please run below command to create a prophecy control plane on a given k8s cluster. 
```
proctl [<customer>] » tenant create -n <controlplane-name> --cluster <cluster-name> --fullname <controlplane-fullname> --email <controlplane-email> --postgres-url <postgres-url> --version 0.4.1-alpha2
```
This command prompts for a password `<controlplane-password>`. This is a long operation and one can track the status of operation with 'controlplane get' command.
Once the deployment status is shown as Deployed, the control plane is said to be deployed successfully.

Creating a control plane is a long operation and one can track the status of operation with 'controlplane get' command. Please use below command to check the status of control plane creation:
```
proctl [<customer>] » tenant get -t <controlplane-name>
```
*Controlplane creation takes around 10 minutes to complete.*


### Data Plane
Data Plane represent an execution environment such as test or production. Please run below command to create a prophecy data plane on a given k8s cluster for a given control plane.
```
Databricks
proctl [<customer>] » dataplane create -n <dataplane-name> -t <tenant-name> --cluster <cluster-name> --fabric-name <fabric-name> --spark-exec-provider databricks --db-org-id <databricks-org-id> --db-token <databricks-token> --db-url <databricks-url> --postgres-url <postgres-url> --version 0.4.1-alpha2

EMR
proctl [<customer>] » dataplane create -n <dataplane-name> -t <tenant-name> --cluster <cluster-name> --fabric-name <fabric-name> --spark-exec-provider emr  --aws-access-key-id <aws-access-key>>--aws-secret-access-key <aws-secret-key>  --emr-prophecy-jar-path <s3-prophecy-jarpath> --emr-log-uri <s3-log-uri> --emr-ec2-subnet-id <subnetid> --postgres-url <postgres-url> --version 0.4.1-alpha2
```

Creating a data plane is a long operation and one can track the status of operation with 'dataplane get' command. Please use below command to check the status of data plane creation:

```
proctl [<customer>] » dataplane get -n <dataplane-name> -t <controlplane-name>
```
*Data plane creation takes around 10 minutes to complete.*


