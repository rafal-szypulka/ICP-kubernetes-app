# How to install [grafana-kubernetes-app](https://github.com/grafana/kubernetes-app) in IBM Cloud Private

Outline of configuration steps:

- Deploy Grafana 5 in ICP using helm chart for Grafana, provided by kubernetes project.
- Modify Grafana Pod with additional container which provide access to cluster API via `kubectl proxy` client. 
- Install [grafana-kubernetes-app](https://github.com/grafana/kubernetes-app).
- Configure connection to cluster API via sidecar container.
- Configure connection to standard ICP Prometheus instance.
- Make minor modifications to Prometheus queries in some of dashboard panels (some labels are specific to ICP).

1). Add `https://kubernetes-charts.storage.googleapis.com` to the list of helm repositories in ICP.

2). Verify that new repository is visible in helm CLI client:

```
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈

$ helm repo list
NAME  	URL
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts
```
3). Configure PersistentVolume where Grafana configuration will be stored.

a. Create file pv-grafana.yaml

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: g5-grafana
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/grafana-data
    type: ""
  persistentVolumeReclaimPolicy: Recycle
```

b. create PersistentVolume g5-grafana using command:

```
kubectl create -f pv-grafana.yml
```

4). Install latest version of Grafana (5.0.4 at the time of writing this procedure).

```
helm install --name g5 stable/grafana --set server.service.type=NodePort –tls
```
5). Verify you can logon to new Grafana instance. 
a.Identify port assigned by NodePort using:

```
$ kubectl get services
(...)
g5-grafana                    NodePort    10.0.0.209   <none>        80:32766/TCP   5d 
```
b. Collect initial admin password for your new Grafana 5 installation using:

```
kubectl get secret --namespace default mm-grafana -o jsonpath="{.data.grafana-admin-password}" | base64 --decode ; echo
```
c. Logon to Grafana `http://<ICP_cluster_ip>:<port>`
where <port> is a port ranging from 30000-32767 collected in step 5a.

6). Verify that PersistentVolume g5-grafana has status BOUND using:

```
kubectl get pv
```
7). Edit configuration of g5-grafana Deployment created during Grafana chart installation.

```
kubectl edit deployment g5-grafana
```
(add the following lines in the `containers:` section):

```
- name: kubectl
        image: k8s.gcr.io/kubectl:v0.18.0-120-gaeb4ac55ad12b1-dirty
        imagePullPolicy: Always
        args: [
          "proxy", "-p", "8001"
        ]
```
8). Login to the Grafana container and install `grafana-kubernetesa-app` from Grafana Labs`

a. Get the name of g5-grafana Pod.

```
kubectl get pod|grep g5-grafana
```
b. Logon to grafana container.

```
kubectl exec -it mm-grafana-599bc546c4-jtv9f -c grafana -- bash
```
c. Install grafana-kubernetes-app.

```
grafana-cli plugins install grafana-kubernetes-app
```
9). Delete g5-grafana Pod to reload Grafana configuration (it will be restored by g5-grafana Deployment).

10). Logon to Grafana and verify that `grafana-kubernetes-app` plugin was installed.

11). Configure connection to ICP cluster, use `http://localhost:8001` as a connection URL and proxy as an access method.

12). Configure connection to default ICP Prometheus instance.

a. Set URL as `https://<prometeus_cluster_ip>:9090`.

b. Set proxy as an access method.

c. Select checkboxes next to `TLS Client Auth`, `With CA Cert` and `Skip TLS Verification (Insecure)`.

d. Copy contents of `<ICP_install_dir>/cluster/cfc-certs/monitoring/ca.pem` to the field `CA Cert`.

e. Copy contents of `<ICP_install_dir>/cluster/cfc-certs/monitoring/client.pem` to the field `Client Cert`.

f. copy contents of `<ICP_install_dir>/cluster/cfc-certs/monitoring/client-key.pem` to the field `Client Key`.

g. Click `Save & Test`.

13). Replace default dashbards for Cluster, Node and Container with dashboards provided in this repository. 







