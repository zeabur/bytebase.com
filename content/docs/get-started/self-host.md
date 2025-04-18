---
title: Self-host Bytebase
---

**Latest release version:** [**%%bb_version%%**](https://github.com/bytebase/bytebase/releases/latest)

Bytebase is a single Go binary and the deployment easy.

## Prerequisites

- Check [System Requirements](/docs/faq##system-requirements).

## Docker

Estimated time: **5 minutes**.

<HintBlock type="info">

If you **run Bytebase inside Docker on Linux** and want to connect the database instance on the same host, then you need to supply the additional `--add-host host.docker.internal:host-gateway --network host` flags.

</HintBlock>

#### Run locally (e.g. localhost:5678)

By default, container listens on [port 80](https://github.com/bytebase/bytebase/blob/main/scripts/Dockerfile#L98). You can overwrite the port by supplying `--port`.

Run the following command to start Bytebase on container port `8080` and map it to localhost port `5678`.

<IncludeBlock url="/docs/get-started/install/terminal-docker-run"></IncludeBlock>

Bytebase will store its data under `~/.bytebase/data` , you can reset all data by running command:

```bash
rm -rf ~/.bytebase/data
```

Check [Server Startup Options](/docs/reference/command-line) for other startup options.

### Use external PostgreSQL to store metadata

Check [Configure External PostgreSQL](/docs/get-started/install/external-postgres) for details.

<IncludeBlock url="/docs/get-started/install/terminal-docker-run-external-url"></IncludeBlock>

### Allow external access via External URL

Run the following command to start Bytebase on port `8080` and map it to localhost port `80`.

```bash
docker run --init \
  --name bytebase \
  --restart always \
  --publish 80:8080 \
  --health-cmd "curl --fail http://localhost:8080/healthz || exit 1" \
  --health-interval 5m \
  --health-timeout 10s \
  --volume ~/.bytebase/data:/var/opt/bytebase \
  bytebase/bytebase:%%bb_version%% \
  --data /var/opt/bytebase \
  --port 8080
```

Follow [Configure External URL](/docs/get-started/install/external-url#configure-via-ui) and then visit Bytebase from
the configured external URL.

### Manifest not found

The docker image only supports linux/amd64 and linux/arm64 arch. If it doesn't match your OS arch, you may supply
`--platform linux/amd64` as a best effort.

### Unable to start using Colima

Due to the vm mechanism of [Colima](https://github.com/abiosoft/colima), try to use the `--mount` option when starting colima as shown below:

```bash
mkdir ~/volumes
colima start --mount ~/volumes:w
docker run --init \
  --name bytebase \
  --restart always \
  --publish 80:8080 \
  --volume ~/.bytebase/data:/var/opt/bytebase bytebase/bytebase:%%bb_version%% \
  --data /var/opt/bytebase \
  --external-url https://bytebase.example.com \
  --port 8080
```

## Kubernetes

Estimated time: **15 minutes**.

### Deploy to Kubernetes

Here is a sample Kubernetes YAML file `bb.yaml` describing the minimal components and configuration required to run Bytebase in Kubernetes.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bytebase
  namespace: default
spec:
  # To prevent data races, only request one replica.
  replicas: 1
  selector:
    matchLabels:
      app: bytebase
  template:
    metadata:
      labels:
        app: bytebase
    spec:
      containers:
        - name: bytebase
          image: bytebase/bytebase:%%bb_version%%
          imagePullPolicy: Always
          env:
            - name: PG_URL
              value: 'postgresql://<<user>>:<<secret>>@<<host>>:<<port>>/<<dbname>>'
          args:
            [
              '--data',
              '/var/opt/bytebase',
              '--external-url',
              'https://bytebase.example.com',
              '--port',
              '8080',
            ]
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: data
              mountPath: /var/opt/bytebase
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 300
            periodSeconds: 300
            timeoutSeconds: 10
      volumes:
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: bytebase-entrypoint
  namespace: default
spec:
  # Optional
  type: LoadBalancer
  selector:
    app: bytebase
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

1. Start Bytebase with the following command:

   ```bash
   kubectl apply -f bb.yaml
   ```

   then you should see output that looks like the following:

   ```plain
   deployment.apps/bytebase created
   service/bytebase-entrypoint created
   ```

2. Make sure everything worked by listing your deployments:

   ```bash
   kubectl get deployments
   ```

   if all is well, your deployment should be listed as follows:

   ```plain
   NAME       READY   UP-TO-DATE   AVAILABLE   AGE
   bytebase   1/1     1            1           10s
   ```

   Do the same check for your services:

   ```bash
   kubectl get services
   ```

   if all is well too, you should see output that looks like the following:

   ```plain
   NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
   bytebase-entrypoint   LoadBalancer   10.100.36.246   localhost     8080:30254/TCP   72s
   kubernetes            ClusterIP      10.96.0.1       <none>        443/TCP          9d
   ```

3. Open a browser and visit [localhost:8080](http://localhost:8080), you should see Bytebase.

#### Upgrade

When a new Bytebase release is published, you can change the image version in the yaml file

```yaml
containers:
  - name: bytebase
    image: bytebase/bytebase:%%bb_version%%
```

Sometimes we need to update the image to the latest digest without changing the image name and version. Or you may want to trigger a restart of all the Bytebase pods without changing the yaml.

In this case, you can run this command:

```bash
kubectl rollout restart deployment/bytebase
```

Kubernetes will rolling restart the pods of the deployment. Because we set `imagePullPolicy: Always`, the new pods will always use the latest image digest.

### Use Helm Chart

#### Production Setup External URL

<HintBlock type="info">

For production setup, you should configure a proper [External URL](/docs/get-started/install/external-url).

</HintBlock>

#### Installing the Chart

```bash
helm -n <YOUR_NAMESPACE> \
--set "bytebase.option.port"={PORT} \
--set "bytebase.option.external-url"={EXTERNAL_URL} \
--set "bytebase.option.pg"={PGDSN} \
--set "bytebase.version"={VERSION} \
install <RELEASE_NAME> bytebase-repo/bytebase
```

For example:

```bash
helm -n bytebase \
--set "bytebase.option.port"=443 \
--set "bytebase.option.external-url"="https://bytebase.example.com" \
--set "bytebase.option.pg"="postgresql://user:secret@foo.ap-east-1.rds.amazonaws.com/postgres" \
--set "bytebase.version"=1.7.0 \
install bytebase-release bytebase-repo/bytebase
```

#### Uninstalling the Chart

```bash
helm delete --namespace <YOUR_NAMESPACE> <RELEASE_NAME>
```

#### Upgrade Bytebase Version/Configuration

Use `helm upgrade` command to upgrade the bytebase version or configuration.

```bash
helm -n <YOUR_NAMESPACE> \
--set "bytebase.option.port"={NEW_PORT} \
--set "bytebase.option.external-url"={NEW_EXTERNAL_URL} \
--set "bytebase.option.pg"={NEW_PGDSN} \
--set "bytebase.version"={NEW_VERSION} \
upgrade bytebase-release bytebase-repo/bytebase
```

### External PostgreSQL

Instead of specify PostgreSQL connection string in helm or Kubernetes yaml file, we allows users to using Kubernetes secrets resources.

#### Kubernetes

Using the following yaml section to replace the `spec.templates.spec.containers.env` section:

```yaml
env:
  - name: PG_URL
    valueFrom:
      secretKeyRef:
        name: secret_name
        key: secrete_key
```

#### Helm

Using `--set bytebase.option.existingPgURLSecret` and `--set bytebase.option.existingPgURLSecretKey` to specify the secret key and secret name instead of `--set "bytebase.option.external-url"={NEW_EXTERNAL_URL}`. See more details in [Bytebase - Artifact Hub](https://artifacthub.io/packages/helm/bytebase/bytebase).

### Persistent Volume

To keep data persistence in production, you need to use the [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes) in the cluster. Each cloud provider has its own solution.

#### For Amazon Elastic Kubernetes Service(EKS)

In AWS EKS, you can use the [Amazon EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) for persistent volumes. Follow the [managing EBS CSI](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html) to add it as an Amazon EKS add-on.

#### For Google Kubernetes Engine(GKE)

Please follow the [Persistent volumes and dynamic provisioning](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes).

## Installation Script

Estimated time: **5 minutes**.

The installation script is stored at [https://github.com/bytebase/install](https://github.com/bytebase/install).

**Prerequisites**

1. Install [curl](https://curl.se/download.html).
2. Install [tar](https://www.gnu.org/software/tar/).

### Install

Using install script to install the latest release version:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/bytebase/install/main/install.sh)"
```

If no error occurs, you should see something like this in the console:

```plain
OS: Darwin
ARCH: arm64
Password:
Get bytebase latest version: %%bb_version%%
Downloading tarball into /var/folders/j4/9x356cb9263f2jryv0xs9pnr0000gn/T/tmp.g1C2PJ8U
Start downloading https://github.com/bytebase/bytebase/releases/download/%%bb_version%%/bytebase_%%bb_version%%_Darwin_arm64.tar.gz...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0
100 81.3M  100 81.3M    0     0  3972k      0  0:00:20  0:00:20 --:--:-- 5430k
Completed downloading https://github.com/bytebase/bytebase/releases/download/%%bb_version%%/bytebase_%%bb_version%%_Darwin_arm64.tar.gz
Start extracting tarball into /opt/bytebase...
Start installing bytebase and bb %%bb_version%%
Installed bytebase %%bb_version%% to /usr/local/bin
Installed bb %%bb_version%% to /usr/local/bin

Check the usage with
  bytebase --help
  bb --help
```

### Run

After install completes, run:

```bash
bytebase --port 8080
```

You should see something like this in the console:

<IncludeBlock url="/docs/get-started/install/terminal-startup-output-success"></IncludeBlock>

#### Troubleshoot

If you encounter any error when you install bytebase by using install script, welcome to open issue on [bytebase/install repository](https://github.com/bytebase/install).

## Build from Source

Estimated time: **30 minutes**.

**Prerequisites**

1. Install [pnpm](https://pnpm.io/installation), Bytebase requires Node.js >=17.0.
2. Install [Go](https://golang.org/dl/), Bytebase requires Go >= 1.16

### Environment Setup

It's recommended to run Bytebase application as non-root user for security reason. If you don't have other non-root users on the system, you can follow the following steps to setup one, e.g. user `bytebase`.

```bash
groupadd bytebase && useradd -g bb bytebase
```

```bash
sudo su bytebase
```

### Build

Download [source code](https://github.com/bytebase/bytebase) from GitHub, then go to the source root directory

<HintBlock type="info">

If you want to build from a specific release `x.y.z`, then switch to that tag.

</HintBlock>

```bash
git checkout tags/x.y.z
```

Build the source

```bash
scripts/build_bytebase.sh [<<out_directory>>]
```

If `out_directory`is not specified, the default directory is `./bytebase-build`

Suppose you run `scripts/build_bytebase.sh foo` After build completes, run:

```bash
foo/bytebase --port 8080
```

(check [Server Startup Options](/docs/reference/command-line) for other startup options)

You should see something like this in the console:

<IncludeBlock url="/docs/get-started/install/terminal-startup-output-success"></IncludeBlock>

**Troubleshoot**

#### error: too many open files

Change the open file limit:

```bash
ulimit -n 10240
```

## Deploy to PaaS

### [Deploy to render](/docs/get-started/install/deploy-to-render/)

### [Deploy to sealos](/docs/get-started/install/deploy-to-sealos/)

### [Deploy to Rainbond](/docs/get-started/install/deploy-to-rainbond/)

### [Deploy to Zeabur](/docs/get-started/install/deploy-to-zeabur/)

## Upgrade from 1.x

If you are upgrading from 1.x, please first upgrade to 2.1.0 and then upgrade to the latest version.
