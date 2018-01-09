## End-to-End Testing of Official Models

In order to facilitate performance and convergence testing of the official Model Garden models, we provide a set of scripts and tools that can:

1. Spin up a Kubernetes cluster on Google Cloud Platform.
2. Deploy jobs to the cluster.
3. Run performance and convergence tests for Garden models.
4. Produce results for easy consumption or dashboard creation.

Want to run your own cluster and tests? Read on!

### Pre-requisites

Note: Throughout this guide, we will assume a working knowledge of [Kubernetes](https://kubernetes.io/) (k8s) and [Google Cloud Platform](https://cloud.google.com) (GCP). If you are new to either, please run through the [appropriate tutorials](https://kubernetes.io/docs/getting-started-guides/gce/) first. (Implied in that: you will need access to a GCP account and project; if you don't have one, [sign up](https://console.cloud.google.com/start).)

You will need [gcloud](https://cloud.google.com/sdk/gcloud/) and [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) installed on your local machine in order to deploy and monitor jobs. Please follow the [Google Cloud SDK installation instructions](https://cloud.google.com/sdk/docs/) and the [kubectl installation instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/). We assume here that you are running Kubernetes v1.9.

After installation, make sure to set gcloud to point to the GCP project you will be using:

```
gcloud config set project <project-id>
```

### Get a Kubernetes cluster

#### Option 1: Bring your own cluster

The tools in the workload directory create pods and jobs on an existing cluster-- which means you need access to a k8s cluster on GCP. The easiest way to do that is to use the [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) (GKE), which has a handy UI. However, GKE does not (as of January 2018) support GPU nodes, so the GKE solution will only work if you are running tests on CPU nodes. 

In any case, if you are using GKE and manually set up a cluster, or if you happen to already have access to a GCP k8s cluster previously created, you can skip the rest of this section on creating a k8s cluster, and instead just configure `gcloud` to access your cluster:

```
gcloud config set project <project-id>
gcloud config set region <region-id>
kubectl config use-context <cluster-auth-id>
```

#### Option 2: Spin up a cluster

If you don't have a cluster already, you can use the scripts in the cluster directory to set one up:

1. In your local copy of the cluster directory, edit the text file cluster/cluster_config.sh to reflect your desired configuration details. (It may be the case that you don't need to change anything at all. If you installed kubectl in a non-standard location, update `KUBE_ROOT`; if you want to use a different region, update `KUBE_GCE_ZONE`, though note that GPUs are only [available in certain regions](https://cloud.google.com/compute/docs/gpus/); if you don't want to use a GPU cluster, update `USE_GPU`; and if you want a different number of worker nodes, update `NUM_NODES`. Note that you should only change the operating system variables if you know what you are doing, as we do not currently support other variants.)

2. All right, this is the wacky part, and I apologize that this is necessary ahead of time. In order to successfully spin up a cluster running Ubuntu, we had to modify the core k8s scripts. I have an [open question on Stack Overflow](https://stackoverflow.com/questions/48121852/kube-up-sh-fails-to-initialize-ubuntu-master-in-cluster-in-kubernetes-v1-9); please feel free to answer that or fix this if you know how. 

For now, you need to add the following to `$KUBE_ROOT/cluster/gce/gci/configure.sh` (most likely for you, that is `$HOME/kubernetes/cluster/gce/gci/configure.sh`), next to all the other functions:

```
function special-ubuntu-setup {
 # Special installation required for ubuntu 16.04, as per Karmel's experience
 # on 2017-12-29
 apt-get install python-yaml

 # Install docker
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
 add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
 apt-get update
 apt-cache policy docker-ce
 apt-get install -y docker-ce
}
```
Then, in "main loop" at the bottom of the same file, call that function before `download-kube-env`. The final main loop should look like this, with the added call to `special-ubuntu-setup` in the middle:

```
######### Main Function ##########
echo "Start to install kubernetes files"
set-broken-motd
KUBE_HOME="/home/kubernetes"
KUBE_BIN="${KUBE_HOME}/bin"
# This is added in by Karmel on 2017-12-29. download-kube-env is 
# failing for lack of yaml, docker, etc.:
special-ubuntu-setup
download-kube-env
source "${KUBE_HOME}/kube-env"
if [[ "${KUBERNETES_MASTER:-}" == "true" ]]; then
  download-kube-master-certs
fi
install-kube-binary-config
echo "Done for installing kubernetes files"
```

3. Now that the painful part is over, you should be able to spin up a cluster with the `setup_cluster.sh` script:

```
cluster/setup_cluster.sh
```
You should see output from k8s indicating status. Note that aqcuiring and setting up GPUs can take several minutes. The final output should indicate that the cluster is up and running, with various API endpoints available; if it doesn't... debug.


### Start your jobs



### Monitoring

You can check in on your cluster using the handy k8s UI. At the command line, run `kubectl proxy`, and then navigate to http://localhost:8001/ui in a browser window. You can authorize yourself by pointing the UI to your k8s config, which is likely `$HOME/.kube/config`. Poke around and make sure your nodes, pods, and jobs are all error free. 