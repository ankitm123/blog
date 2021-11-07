---
title: Running JenkinsX v3 locally on k3s
tags: ['JenkinsX', 'golang', 'k3s', 'v3', 'vault']
date: '2021-11-07'
---

With so many CI/CD solutions available, it would be nice if we can do a quick POC.
This guide is exactly that:

**Install JenkinsX v3 on a local single node k3s cluster with secrets stored externally in a vault running in a docker container in less than 30 minutes (No money back gurantees though if it takes longer than that)!**

> Disclaimer:
> This is my local set up for quickly debugging JenkinsX issue.
> Do not use this in production!

#### Install k3s

I used the script to install it quickly using the command

```bash
curl -sfL https://get.k3s.io | sh -
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/k3s-config
```

Detailed instructions to install k3s are [here](https://rancher.com/docs/k3s/latest/en/installation/)

Once installed, you should be able to run:

```bash
kubectl get nodes
NAME                STATUS   ROLES                  AGE   VERSION
<k3s node name>     Ready    control-plane,master   78d   v1.21.5+k3s2
```

The name of the node will be used when setting the vault_addr when running jx admin operator.

#### Set up Vault

Install the [vault cli](https://www.vaultproject.io/docs/install).

Run vault in a docker container locally

```bash
docker run --cap-add=IPC_LOCK -p 8200:8200 -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' --net host vault:latest
```

In another terminal, set the VAULT_ADDR

```bash
export VAULT_ADDR='http://0.0.0.0:8200'
```

Enable kubernetes auth.

```bash
vault auth enable kubernetes
```

After we set up JenkinsX in the k3s cluster, we will configure the vault kubernetes backend.

#### Install JX v3

Use the [jx3-k3s template](https://github.com/ankitm123/jx3-k3s), and generate a cluster git repository by clicking [here](https://github.com/ankitm123/jx3-k3s/generate).
Clone the newly created repository.
Retrieve the name of the k3s node by running `kubectl get nodes`.

Edit the `jx-requirements.yaml` and set the value of vault url to `"http://<replace with k3s node name>:8200"`

Commit and push the changes to the cluster git repository:

```bash
git add .
git commit -m "fix: set vault url"
git push origin main
```

Set the `GIT_USERNAME` and `GIT_TOKEN` env variable and run:

```bash
jx admin operator --username $GIT_USERNAME --token $GIT_TOKEN --url <url of the cluster git repo> --set "jxBootJobEnvVarSecrets.EXTERNAL_VAULT=\"true\"" --set "jxBootJobEnvVarSecrets.VAULT_ADDR=http://<replace with k3s node name>:8200"
```

The first job will fail as it cannot authenticate against vault.
Once the `secret-infra` namespace has been created, we can configure the kubernetes backend

#### Vault set up

The instructions are taken from [here](https://learn.hashicorp.com/tutorials/vault/kubernetes-external-vault?in=vault/kubernetes#configure-kubernetes-authentication).

Configure the vault kubernetes backend:

```bash
VAULT_HELM_SECRET_NAME=$(kubectl -n secret-infra get secrets --output=json | jq -r '.items[].metadata | select(.name|startswith("kubernetes-external-secrets-token-")).name')
TOKEN_REVIEW_JWT=$(kubectl -n secret-infra get secret $VAULT_HELM_SECRET_NAME --output='go-template={{ .data.token }}' | base64 --decode)
KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')
vault write auth/kubernetes/config \
        token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
        kubernetes_host="$KUBE_HOST" \
        kubernetes_ca_cert="$KUBE_CA_CERT" \
        disable_iss_validation=true
```

Create a vault role:

```bash
vault write /auth/kubernetes/role/jx-vault bound_service_account_names='*' bound_service_account_namespaces=secret-infra token_policies=jx-policy token_no_default_policy=true disable_iss_validation=true
```

Create the policy attached to the role:

```bash
vault policy write jx-policy - <<EOF
path "secret/*" {
  capabilities = ["sudo", "create", "read", "update", "delete", "list"]
}
EOF
```

#### Set up ingress and webhook

Get the external IP of the traefik service (loadbalancer)

```bash
kubectl get svc -A | grep LoadBalancer
kube-system   traefik          LoadBalancer   10.43.103.73    <external-ip>    80:31123/TCP,443:31783/TCP   40m
```

Edit the jx-requirements.yaml file by editing the ingress domain:

```bash
jx gitops requirements edit --domain <external-ip>.nip.io
```

Next, download and install [ngrok](https://ngrok.com/download).
Run this in a new terminal window/tab:

```bash
jx ns jx
kubectl port-forward svc/hook 8090:80
```

```bash
ngrok http 8090
```

Once this tunnel is open, paste the ngrok url in the hook field in the `helmfiles/jx/jxboot-helmfile-resources-values.yaml` file in the cluster git repository.

commit and push the changes.

```bash
git add .
git commit -m "chore: new ngrok ip"
git push origin main
```

Once the boot job has succeeded, you should see:

```bash
HTTP Requests
-------------

POST /hook                     200 OK
```

in the ngrok terminal.

In a subsequent blog, we can look at [importing projects](https://jenkins-x.io/v3/develop/create-project/) into our jenkinsX installation.

Thanks to Steve Melo and Dave C in the _jenkins-x-user_ slack channel for great feedback and document contributions!
