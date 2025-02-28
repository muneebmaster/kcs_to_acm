# Config Sync to ACM Migration

**Important:** This is not the official google documentation and not supposed to be treated as such.

## Comparison Config Sync vs ACM
[Config Sync](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync/overview) allows cluster operators to manage single clusters, multi-tenant clusters, and multi-cluster Kubernetes deployments using files, called configs, stored in a Git repository. It has the following features:
    
    * Git-syncing functionality (Single, Multi, Unstructured repo's)
    * Hierarchy Controller (HNC)
    * Installed via `config-sync-operator.yaml`
    * Free tool for any GKE customer and intended to run on GKE

[Anthos Config Management](https://cloud.google.com/anthos/config-management) (ACM) enables configuration + policy distribution and hierarchical policy enforcement for multi-tenant, multi-cluster Kubernetes deployments. It has the following differences compared to Config Sync:

    * Installed via `config-management-operator.yaml` 
    * Enables Policy Control over configurations
    * Policy enforcement via Tested Gatekeeper Deployment 
    * Provides catalog of Gatekeeper constraints
    * Requires a valid Anthos license
    * Can be installed and observed through Anthos UI


## Migration from Config Sync to ACM

**Assumption 1:** It is assumed that Config Sync Operator has been deployed and ACM repository has been configured

**Assumption 2:** It is assumed that GKE clusters have been registered in the [Anthos Hub](https://cloud.google.com/anthos/multicluster-management/connect/registering-a-cluster#register_cluster) and show status as `Synced`

To verify if a cluster is already registered check Anthos UI or run the following command:

```
gcloud container hub memberships list
```


**Step 1** Enable Anthos Config Management by completing the following steps. 
[Reference 1](https://cloud.google.com/anthos-config-management/docs/how-to/installing#enabling)

```
 gcloud alpha container hub config-management enable
```

**Step 2** Download ACM Operator
[Reference 2](https://cloud.google.com/anthos-config-management/docs/how-to/installing-config-sync#configuring-config-sync)

```
gsutil cp gs://config-management-release/released/latest/config-management-operator.yaml config-management-operator.yaml
```

Note: This will fetch the latest version of config-management-operator 1.6.2. If you need to use specific version of ACM, please
download it [here](https://cloud.google.com/anthos-config-management/downloads#v162)


**Step 3** Deploy ACM Operator
[Reference 2](https://cloud.google.com/anthos-config-management/docs/how-to/installing-config-sync#configuring-config-sync)

```
kubectl apply -f config-management-operator.yaml
```

Monitor Rolling Update here:
```
kubectl get pods -n kube-system -w | grep config-management-operator
```


!!! result
    As a result of this update Config Sync Image will be replaced by Config-management one via rolling upgrade.


**Step 4** Verify that ACM Synchronized with Git Repo

In GCP UI go to Anthos -> Config Management -> Cluster, you should see that migrated cluster has status `Synced` for `Config sync status`


(Optional) CLI Method:

```
nomos status
```

!!! result
    Output shows that cluster is `Synced`

**Output:** 
```
*gke_sem-demo-march_us-central1-b_ayratk-cluster
  --------------------
  <root>   git@github.com:repo/youname.git@main
  SYNCED   fe46984e
```

**Step 5** Verify that new resources are propagating to the Cluster.

Create a `test` namespace manifest resource in `namespace` folder of the Git Repo.

```
cd namespaces
cat <<EOF > namespace.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: test
EOF
```

```
git add .
git commit "test naemspaces"
git push
```

!!! result
    `test` Namespaces has been created.


## Enable ACM Policy Controller Configuration

**Step 1** Enable ACM Policy Configuration
[Reference 3](https://cloud.google.com/anthos-config-management/docs/how-to/installing-policy-controller#installing)


Update `config-management.yaml` with Policy controller configuration for the migrated cluster

```
# config-management.yaml

apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  # Set to true to install and enable Policy Controller
  policyController:
    enabled: true
```

Apply configuration:

```
kubectl apply -f config-management.yaml
```

**Step 2** Verify Gatekeeper Installation


**UI Method:**

In GCP UI go to Anthos -> Config Management -> Cluster, you should see that migrated cluster has status `Installled` for `Policy controller status`


**kubectl method:**

```
kubectl get pods -n gatekeeper-system
```

**Output:**
```
NAME                                             READY   STATUS    RESTARTS   
gatekeeper-audit-5f867bc644-xw7q6                1/1     Running   0          
gatekeeper-controller-manager-57d8699bdc-dfxwl   1/1     Running   0          
```


!!! result
    `gatekeeper-audit` and `gatekeeper-controller-manager` are running 



**gcloud method:**

```
gcloud alpha container hub config-management status \
    --project=PROJECT_ID
```

**Output:**

```
Name                Status  Last_Synced_Token  Sync_Branch  Last_Synced_Time      Policy_Controller
my-cluster  SYNCED  fe46984                     main         2021-03-10           INSTALLED
```

**Step 3** (Optional) Create Test Policy allow repo, that only allows to deploy from gcr.io

If you used `nomos init` to create you Config Sync repo folder structure you should have something like similar folder tree view:

```
├── README.md
├── cluster
├── clusterregistry
├── namespaces
│   ├── namespace.yaml
└── system
    ├── README.md
    └── repo.yaml
```

Policies can be stored in `cluster` folder so they can be applied cluster wide.

```
cd clusters
```

```
cat <<EOF > allowed-repos.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    repos:
      - "gcr.io"
      - "gke.gcr.io"
EOF
```

Commit code to GitHub and wait for synchronization

```
git add .
git commit "create test policy"
git push
```

**Step 4** (Optional) Deploy nginx container from `dockerhub`

```
kubectl run nginx --image=nginx -n test
```

**Output:**

```
Error from server ([denied by allowed-repos] container <nginx> has an invalid image repo <nginx>, allowed repos are ["gcr.io", "gke.gcr.io"]): admission webhook "validation.gatekeeper.sh" denied the request: [denied by allowed-repos] container <nginx> has an invalid image repo <nginx>, allowed repos are ["gcr.io", "gke.gcr.io"]
```


!!! result
    `Nginx` deployed has been blocked as it was deployed from Docker hub and not via GCR. Our Policy controller  is working as expected.
