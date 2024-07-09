---
slug: switching-the-storage-class
id: wyxtky6wpthz
type: challenge
title: Switching the storage class
teaser: This section goes through the process for changing the storage class used
  by Redpanda.
notes:
- type: text
  contents: |-
    Now we continue to our first cluster operation: changing the storage class used by Redpanda.

    ![skate.png](../assets/skate.png)
tabs:
- title: Shell
  type: terminal
  hostname: server
- title: Prometheus
  type: service
  hostname: server
  path: /
  port: 9090
- title: Grafana
  type: service
  hostname: server
  path: /
  port: 3000
- title: Producer
  type: terminal
  hostname: server
  cmd: ./produce.sh
difficulty: basic
timelimit: 600
---

# Introduction

This scenario focuses on changing the storage class used by a running Redpanda cluster. This process involves updating the operator and StatefulSet, setting up an additional broker to ensure replica factor is maintained throughout the process, and then deleting/decommissioning the original pods while managing the PersistentVolumes.

Changing the broker count should involve a number of considerations:

- Availability: do you have enough brokers to span across all racks and/or availability zones?
- Cost: infrastructure costs will be impacted by a change in broker count
- Data retention: storage capacity and possible retention values are determined in large part by the local disk capacity across all brokers
- Durability: you should have more brokers than your lowest partition replication factor
- Partition count: this value is determined primarily by the CPU core count of the overall cluster

See [our docs](https://docs.redpanda.com/current/manage/kubernetes/decommission-brokers/) for more details on the above considerations.

---

# Step 1) Add a New K8s Node (Optional)

Since the storage class change process involves using an additional broker, it may be necessary to add additional Kubernetes capacity.

However, in this tutorial, we already have a 4th node available for use:

```bash,run
kubectl get nodes
```

Output:

```bash,nocopy
NAME      STATUS   ROLES                  AGE   VERSION
worker1   Ready    <none>                 27m   v1.28.6+k3s2
worker2   Ready    <none>                 26m   v1.28.6+k3s2
worker3   Ready    <none>                 26m   v1.28.6+k3s2
worker4   Ready    <none>                 25m   v1.29.0+k3s1
server    Ready    control-plane,master   30m   v1.29.0+k3s1
```

---

# 2) Install the new storage class

In your environment, install an appropriate storage class for use. For most users, the [LVM CSI Driver](https://github.com/metal-stack/csi-driver-lvm) from Metal Stack is a great choice, as noted in our [documentation](https://docs.redpanda.com/current/deploy/deployment-option/self-hosted/kubernetes/gke-guide/#create-sc).

However, for this tutorial, we will simply create an alternate `local-path` driver.

Firstly, we get a copy of the current storage class. Then we rename it and set `is-default-class` to false:

```bash,run
kubectl get storageclass -A local-path -o yaml > sc.yaml
sed -i 's/name: local-path/name: local-path-alt/' sc.yaml
sed -i 's/is-default-class: "true"/is-default-class: "false"/' sc.yaml
cat sc.yaml
```

Output:

```bash,nocopy
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    objectset.rio.cattle.io/applied: H4sIAAAAAAAA/4yRT+vUMBCGv4rMua1bu1tKwIOu7EUEQdDzNJlux6aZkkwry7LfXbIqrIffn2PyZN7hfXIFXPg7xcQSwEBSiXimaupSxfJ2q6GAiYMDA9/+oKPHlKCAmRQdKoK5AoYgisoSUj5K/5OsJtIqslQWVT3lNM4xUDzJ5VegWJ63CQxMTXogW128+czBvf/gnIQXIwLOBAa8WPTl30qvGkoL2jw5rT2V6ZKUZij+SbG5eZVRDKR0F8SpdDTg6rW8YzCgcSW4FeCxJ/+sjxHTCAbqrhmag20Pw9DbZtfu210z7JuhPnQ719m2w3cOe7fPof81W1DHfLlE2Th/IEUwEDHYkWJe8PCsgJgL8PxVPNsLGPhEnjRr2cSvM33k4Dicv4jLC34g60niiWPSo4S0zhTh9jsAAP//ytgh5S0CAAA
    objectset.rio.cattle.io/id: ""
    objectset.rio.cattle.io/owner-gvk: k3s.cattle.io/v1, Kind=Addon
    objectset.rio.cattle.io/owner-name: local-storage
    objectset.rio.cattle.io/owner-namespace: kube-system
    storageclass.kubernetes.io/is-default-class: "false"
  creationTimestamp: "2024-07-09T09:14:35Z"
  labels:
    objectset.rio.cattle.io/hash: 183f35c65ffbc3064603f43f1580d8c68a2dabd4
  name: local-path-alt
  resourceVersion: "290"
  uid: 180e4124-e140-4edf-9afa-a233f66cf1df
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Now we install our new driver and validate that it exists:

```bash,run
kubectl create -f sc.yaml
kubectl get sc
```

Output

```bash,nocopy
storageclass.storage.k8s.io/local-path-alt created
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  21m
local-path-alt         rancher.io/local-path   Delete          WaitForFirstConsumer   false                  9m58s
```

Notice that we now have `local-path-alt` ready to use. It still uses the same provisioner `rancher.io/local-path`.

Next, we can show what persistent volume claims already exist and what storage class they are using:

```bash,run
# Show the PVCs and storage classes in use
kubectl get pvc -n redpanda -o yaml | yq '.items[] | "\(.metadata.name)    \(.spec.storageClassName)"'
```

Output:

```bash,nocopy
datadir-redpanda-0    local-path
datadir-redpanda-1    local-path
datadir-redpanda-2    local-path
```

---

# 3) Change the default storage class

In order for the new storage class to be used, both the operator and the statefulset need to have the class defined. The storageclass is only used during the creation of a persistent volume claim, therefore simply changing the class within the cluster definition isn't sufficient: we also need the change to be pushed into the statefulset.

To do this, we will delete the statefulset but choose to do so while orphaning the existing pods - this will leave the pods in place. With the statefulset removed, we can then patch the cluster definition, which will force the recreation of the statefulset:

```bash,run
# Force the removal of the sts to prevent auto re-creation of the pod / pvc
kubectl delete sts redpanda --cascade=orphan -n redpanda

# Create patch file with storage class change
cat <<EOF > patch.yaml
spec:
  clusterSpec:
    statefulset:
      replicas: 4
    storage:
      persistentVolume:
        storageClass: local-path-alt
EOF

# Patch the operator deployment with the change
kubectl patch redpanda redpanda -n redpanda --type merge --patch-file patch.yaml

# Watch for the deployment to be reconciled
kubectl get redpanda -n redpanda --watch
```

Once the reconciliation shows as succeeded, press Ctrl-C to continue.

Output:

```bash,nocopy
statefulset.apps "redpanda" deleted
redpanda.cluster.redpanda.com/redpanda patched
NAME       READY   STATUS
redpanda   False   HelmRelease 'redpanda/redpanda' is not ready
redpanda   True    Redpanda reconciliation succeeded
```

Next, validate the health of the cluster and show there is an additional broker:

```bash,run
rpk cluster health
```

Output:

```bash,nocopy
CLUSTER HEALTH OVERVIEW
=======================
Healthy:                          true
Unhealthy reasons:                []
Controller ID:                    0
All nodes:                        [0 1 2 3]
Nodes down:                       []
Leaderless partitions (0):        []
Under-replicated partitions (0):  []
```

Notice that there are 4 broker IDs listed (`All nodes: [0 1 2 3]` ) and the cluster is marked as healthy.

```bash,run
# Show the PVCs and storage classes in use
kubectl get pvc -n redpanda -o yaml | yq '.items[] | "\(.metadata.name)    \(.spec.storageClassName)"'
```

Output:

```bash,nocopy
datadir-redpanda-0    local-path
datadir-redpanda-1    local-path
datadir-redpanda-2    local-path
datadir-redpanda-3    local-path-alt
```

Notice also that the new broker is already using our new storage class!

At this point, both the cluster definition and the statefulset are aware of the new storage class - and as we've seen, any new brokers will use the new storage. Great!

But the existing brokers are unchanged - for these to change as well, we now need to decommission them and move their workloads to the new broker - which is why we readied that in Step 1.

---

# 4) Manual Rolling Restart

### For broker ID 0:

```bash,run
rpk redpanda admin brokers decommission 0

until rpk redpanda admin brokers decommission-status 0 | grep -q "Node 0 is decommissioned successfully.";
do
  echo "Waiting for decommission to succeed.."
  sleep 1;
done
```

```bash,run
PV=$(kubectl get pv -n redpanda | grep datadir-redpanda-0 | awk '{print $1}')
kubectl patch pv -n redpanda $PV -p '{"metadata":{"finalizers":null}}'
kubectl patch pvc -n redpanda datadir-redpanda-0 -p '{"metadata":{"finalizers":null}}'

kubectl delete pvc -n redpanda datadir-redpanda-0 &
kubectl delete pod -n redpanda redpanda-0
```

```bash,run
rpk cluster health
kubectl get pvc -n redpanda -o yaml | yq '.items[] | "\(.metadata.name)    \(.spec.storageClassName)"'
```

### For broker ID 1:

```bash,run
rpk redpanda admin brokers decommission 1

until rpk redpanda admin brokers decommission-status 1 | grep -q "Node 1 is decommissioned successfully.";
do
  echo "Waiting for decommission to succeed.."
  sleep 1;
done
```

```bash,run
PV=$(kubectl get pv -n redpanda | grep datadir-redpanda-1 | awk '{print $1}')
kubectl patch pv -n redpanda $PV -p '{"metadata":{"finalizers":null}}'
kubectl patch pvc -n redpanda datadir-redpanda-1 -p '{"metadata":{"finalizers":null}}'

kubectl delete pvc -n redpanda datadir-redpanda-1 &
kubectl delete pod -n redpanda redpanda-1
```

```bash,run
rpk cluster health
kubectl get pvc -n redpanda -o yaml | yq '.items[] | "\(.metadata.name)    \(.spec.storageClassName)"'
```

### For broker ID 2:

```bash,run
rpk redpanda admin brokers decommission 2

until rpk redpanda admin brokers decommission-status 2 | grep -q "Node 2 is decommissioned successfully.";
do
  echo "Waiting for decommission to succeed.."
  sleep 1;
done
```

```bash,run
PV=$(kubectl get pv -n redpanda | grep datadir-redpanda-2 | awk '{print $1}')
kubectl patch pv -n redpanda $PV -p '{"metadata":{"finalizers":null}}'
kubectl patch pvc -n redpanda datadir-redpanda-2 -p '{"metadata":{"finalizers":null}}'

kubectl delete pvc -n redpanda datadir-redpanda-2 &
kubectl delete pod -n redpanda redpanda-2
```

```bash,run
rpk cluster health
kubectl get pvc -n redpanda -o yaml | yq '.items[] | "\(.metadata.name)    \(.spec.storageClassName)"'
```

---

# 5) Remove additional broker (Optional)

At this point, the cluster is now fully running on the new storage class. If the additional broker we added earlier should be removed, we can take similar steps.

Firstly, we need to make sure we have connectivity through to the additional broker, since our admin commands will need to be sent there directly. In the demo environment, this means adding a hostname entry to /etc/hosts, but this will likely differ in your cluster.

Secondly, we need to add the extra broker to the list of admin endpoints in the rpk profile.

```bash,run
IP=$(kubectl get services -n redpanda | grep redpanda-3 | awk '{print $4}')
echo "$IP redpanda-3.testdomain.local" >> /etc/hosts

rpk profile set admin.hosts=redpanda-0.testdomain.local:31644,redpanda-1.testdomain.local:31644,redpanda-2.testdomain.local:31644,redpanda-3.testdomain.local:31644
```

With connectivity in place, we can now begin the decommissioning process.

Firstly we disable the STS in order to better control exactly which broker is removed - again being sure to use the orphan cascade option:

```bash,run
# Force the removal of the sts to prevent auto re-creation of the pod / pvc
kubectl delete sts redpanda --cascade=orphan -n redpanda
```

Next, we decommission the broker we added. Before we started the cluster consisted of IDs 0, 1 and 2. The additional broker we added had an ID of 3.

```bash,run
rpk redpanda admin brokers decommission 3

until rpk redpanda admin brokers decommission-status 3 | grep -q "Node 3 is decommissioned successfully.";
do
  echo "Waiting for decommission to succeed.."
  sleep 1;
done
```

Once successfully decommissioned, we can delete the persistent volume claim and the pod:

```bash,run
PV=$(kubectl get pv -n redpanda | grep datadir-redpanda-3 | awk '{print $1}')
kubectl patch pv -n redpanda $PV -p '{"metadata":{"finalizers":null}}'
kubectl patch pvc -n redpanda datadir-redpanda-3 -p '{"metadata":{"finalizers":null}}'

kubectl delete pvc -n redpanda datadir-redpanda-3 &
kubectl delete pod -n redpanda redpanda-3
```

Next, we validate that the cluster is healthy:

```bash,run
rpk cluster health
```

Once the cluster is showing as healthy, we patch the cluster definition to make sure the cluster is defined with the correct number of brokers. In this tutorial, we started with 3 brokers:

```bash,run
# Create patch file with storage class change
cat <<EOF > patch.yaml
spec:
  clusterSpec:
    statefulset:
      replicas: 3
EOF

# Patch the operator deployment with the change
kubectl patch redpanda redpanda -n redpanda --type merge --patch-file patch.yaml

# Watch for the deployment to be reconciled
kubectl get redpanda -n redpanda --watch
```

Once the operator has reconciled the cluster, our process is complete.

Time for more coffee!