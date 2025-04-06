# Soda Agent in Kubernetes

Janne Kauhanen 2025-04-06

We will create

* A local Minikube Kubernetes cluster that runs a demo database and a Soda Agent, connected in the cluster
* A connection between the agent and Soda Cloud
* A database connection to `soda` CLI running on laptop
* A Github runner that runs checks on the database as part of Github Action jobs

## Minikube using Podman in Windows Subsystem for Linux (WSL)

* For easier installation of Podman, use Ubuntu >= 20.10 (tested Ubuntu 24.04.1 LTS)
* [Install Podman](https://podman.io/docs/installation), version should be >= 4.9.0
* [Install kubectl](https://k8s-docs.netlify.app/en/docs/tasks/tools/install-kubectl/)
* [Install Helm](https://helm.sh/docs/intro/install/)
* [Install minikube](https://k8s-docs.netlify.app/en/docs/tasks/tools/install-minikube/)
  * In WSL there is no need to worry about Hypervisor, just run
    ```
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    ```
* Minikube is trying to run Podman with elevated privileges, but it can't do that silently (sudo -n) because your system still prompts for a password. You now have two valid options - pick one depending on what you prefer:
  1. `sudo visudo`, add `<your-username> ALL=(ALL) NOPASSWD: /usr/bin/podman` (tested this one)
  2. `minikube config set rootless true`
* When testing, minikube had difficulties loading images. This was an `resolv.conf` issue:
  * Remove the symbolic link `/etc/resolv.conf` and replace it with the static file with the following content (Google DNS)
    ```
    nameserver 8.8.8.8
    ```
  * Add
    ```
    [network]
    generateResolvConf = false
    ``` 
    into `/etc/wsl.conf` to prevent automatic update of `resolv.conf`.
  * Restart WSL: in DOS prompt, `wsl --shutdown`
* Start minikube and make sure kubectl is connected to it:

  ```
  minikube start --driver=podman
  kubectl config get-contexts
  kubectl config use-context minikube
  ```

## Soda Client

This workflow was tested using Anaconda:

```
conda create -n soda python=3.10
conda activate soda 
```

The Soda CLI:

```
pip install -i https://pypi.cloud.soda.io soda-postgres
pip install -i https://pypi.cloud.soda.io soda-scientific
```

## Soda Agent

To install a Soda Agent in the Kubernetes cluster, create a new [Soda Cloud agent](https://docs.soda.io/soda-agent/deploy.html#create-a-soda-cloud-account) and copy `soda.apikey` and `soda.apikey.secret` into a yaml file:

`soda-helm-local-values.yml`:

```
soda:
   apikey:
     id: "xxxxxxxxxx"
     secret: "xxxxxxxxxxxxxx"
   agent:
     name: "soda-agent-local-kube"
     logformat: "raw"
     loglevel: "ERROR"
   cloud:
     # Use https://cloud.us.soda.io for US region; use https://cloud.soda.io for EU region
     endpoint: "https://cloud.soda.io"
```

```
helm repo add soda-agent https://helm.soda.io/soda-agent/
helm install soda-agent soda-agent/soda-agent --values soda-helm-local-values.yml

helm list -A
helm status soda-agent
helm get all soda-agent
```

If success, the `soda-agent-local-kube` should appear (may take several minutes) in Soda Cloud > avatar > Agents.

Cleanup:
```
helm uninstall soda-agent
```

## The database

Install the database into the Kubernetes cluster:

`soda-db.yml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: soda-database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: soda-database
  template:
    metadata:
      labels:
        app: soda-database
    spec:
      containers:
      - name: soda-database
        image: sodadata/soda-adventureworks
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: xxxxxx
#          valueFrom:
#            secretKeyRef:
#              name: soda-db-secret
#              key: POSTGRES_PASSWORD
---
apiVersion: v1
kind: Service
metadata:
  name: soda-database
spec:
  selector:
    app: soda-database
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
  type: ClusterIP
```

```
kubectl apply -f soda-db.yml
kubectl get pods
```

Port-forward to localhost

* to run scans in the Kubernetes cluster using `soda` on laptop
* to enable Github runner to access the database

```
kubectl port-forward svc/soda-database 5432:5432
```

Test the connection and run scans:

```
soda test-connection --data-source localkube --configuration local_kube_configuration.yml
soda scan --data-source localkube --configuration local_kube_configuration.yml --template-path templates.yml checks.yml
```

The database is visible also in Windows localhost (DBeaver) without any complications. When running the cluster in Windows Docker/Podman, seeing localhost in WSL Ubuntu requires some effort.

To connect the agent to the data source, go to Soda Cloud > avatar > Data Sources, New Data Source. In Connect tab, the host name is now our service `soda-database`:

```
data_source localkube:
  type: postgres
  connection:
    host: soda-database
    port: 5432
    username: xxxx
    password: xxxx
    database: postgres
  schema: public
```

## Github runner

[Soda documentation](https://docs.soda.io/soda/quick-start-dev.html#create-a-github-action-job)

[Github documentation](https://github.com/jpkau/sodatest/settings/actions/runners/new)

* In Github, choose Actions > Runners > Self-hosted runners > New runner, Linux
* The required code is provided
* Start the runner (`./run.sh`) in an environment where the soda CLI is installed
* This runner automatically takes jobs tagged `self-hosted` or `linux`
