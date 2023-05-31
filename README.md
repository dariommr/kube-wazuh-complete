# Wazuh Environment complete manifest files
This environment is meant to be used in a local Kubernetes cluster.

## Pre-requisites
### Kubernetes local cluster
This guide could help you to install the local cluster: [Installation Simple Kubernetes Cluster on Ubuntu Server 20.04](https://achchusnulchikam.medium.com/installation-simple-kubernetes-cluster-on-ubuntu-server-20-04-91e4868558b7)

### Prepare the Storage
As we are running a local Kubernetes cluster, we need to prepare the storage and configure Kubernetes for the StorageClass and persistent volumes.

**Create the folders**

```shell
sudo mkdir -p /mnt/kube-storage/mnt1
sudo mkdir -p /mnt/kube-storage/mnt2
sudo mkdir -p /mnt/kube-storage/mnt3
sudo mkdir -p /mnt/kube-storage/mnt4
```

**Modify the StorageClass**

If you are working from this repository, it is not needed to modify it.
If you are using the [Wazuh oficial repository](https://github.com/wazuh/wazuh-kubernetes/tree/v4.4.1), you need to modify the StorageClass to match this:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: wazuh-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

**Create the Persistent Volumes**

Since we are working with a Kubernetes local cluster, the persistent volumes will not be created and assigned dynamically ([See documentation](https://kubernetes.io/docs/concepts/storage/storage-classes/#local)), so we need to create them.
To create the Persistent Volumes run this command:
```shell
kubectl apply -f envs/local-env/create-pvs.yaml
```

## Additional Pods
To have a complete Wazuh Environment with all the features working, such as Load balancing and SMTP relay, I added two pods to achieve this:
- Nginx Load Balancer Pod
- Postfix SMTP relay (based on Gmail)

### Nginx Pod Configuration
To make use of this Pod, first we need to set some things up

**Create the SSL Certificates**

```shell
openssl req -out nginx-access.pem -new -x509
```
Create a passphrase and complete the information requested.
Once created, move the certificate files `nginx-access.pem` and `privkey.pem` to the `wazuh/nginx/nginx_conf` folder.

**Modify the Nginx Secret**

Before modify the secret you need to convert the password to a base64 string:
```shell
echo <YOUR_PASSPHRASE> | base64
```
Once is converted you need to modify the file `wazuh/secrets/nginx-secret.yaml`:
```yaml
data:
  nginx-access.pass: |
    xxxxxxxxxxxxxx                   <--- base64 encoded Passphrase
```

### SMTP Pod Configuration

This pod is launched from a container I created and you can find it here: [dariommr/wazuh-postfix](https://hub.docker.com/repository/docker/dariommr/wazuh-postfix/general)

If you need to build your own container, here you can find a Dockerfile example: [dariommr/dockerfiles/Postfix](https://github.com/dariommr/dockerfiles/tree/main/Postfix)

**Modify the SMTP Secret**

At this point, you need to do the same as before, convert your email address and the GMail App password to base64 encoded strings:
```shell
echo <YOUR_EMAIL> | base64
echo <YOUR_GMAIL_PASS> | base64
```

Once you have this data, you can proceed to modify the secret (`wazuh/secrets/smtp-secret.yaml`).
```yaml
data:
  email: xxxxxxxxxxxx== #base64 encoded
  password: xxxxxxxxx== #base64 encoded
```

## Run the environment
### Setup SSL Certificates
Before running the environment, you need to create the SSL Certificates for Wazuh Indexer and Dashboard. Please follow [this guide](https://documentation.wazuh.com/current/deployment-options/deploying-with-kubernetes/kubernetes-deployment.html#setup-ssl-certificates)
>You can generate self-signed certificates for the Wazuh indexer cluster using the script at `wazuh/certs/indexer_cluster/generate_certs.sh` or provide your own.
>You can generate self-signed certificates for the Wazuh dashboard cluster using the script at `wazuh/certs/dashboard_http/generate_certs.sh` or provide your own.

### Apply the manifests
Once you have all set up, you can run the environment by issuing this command:
```shell
kubectl apply -k envs/local-env/.
```

## Additional information
### Ports information
| PORT  |             SERVICE               |
| ----- | --------------------------------- |
| 31443 | Wazuh Dashboard (UI) Access       | 
| 31514 | Wazuh Agent-Manager communication |
| 31515 | Wazuh Agent Registration          |

### API Access
The access to the APIs will be through the same port as the Wazuh Dashboard but with different endpoint.
- Wazuh Manager API: `https://<DASHBOARD_IP>:31443/api/wazuh`
- Wazuh Indexer API: `https://<DASHBOARD_IP>:31443/api/elastic`
