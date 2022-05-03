ISTIO With Kustomize

This walks through one way of using customize to update the Anchore deployment to prevent Istio sidecars from being injected into Job pods.
There are other ways this can be accomplished.

There are 2 overlay directories here which represent two different methods of patching.  This readme doesn't not cover the mesh/istio-kill overlay, only the mesh/jsonpatch option.

JSON Patch was used here versus Strategic Merge Patch the amount of patching to be done was very minimal (1 line in multiple files). 

Additionally, with the mesh/istio-kill option, the Strategic Merge Patch did not appear to be a good fit because kustomize doesn't seem to handle case very well 
(appending to the container cmd args list) because the patch directive only works when the merge key for the array is defined in k8s API 
(https://github.com/kubernetes-sigs/kustomize/issues/3265)

## Istio Setup
Install istio in your cluster
https://istio.io/latest/docs/setup/getting-started/

curl -L https://istio.io/downloadIstio | sh -
cd istio-1.12.1
export PATH=$PWD/bin:$PATH (or add it your path)

Then...
In your cluster you need to label the namespace you want istio to operate on
kubectl label namespace <your namespace here> istio-injection=enabled

Now when you deploy to your cluster/namespace the istio proxy will get injected into every pod. 

## Kustomize setup
For the purposes of this I created a directory structure to help keep the overlays separated, so
k8s
  -> base      (this is where the base anchore-deploy.yaml file exists, this is the deployment file output from the values.yaml and helm templates (more on this later))
  -> overlays  (this is where the patch files live, that is the changes we want to make)

For our purposes we will have 

k8s->overlays->jsonpatch->istio-prevent.json  (in this file we define what we want to inject into our deployment...that 'patch')
k8s->overlays->jsonpatch->kustomization.yaml  (this has the "instructions" for the kustomization)

We'll come back to these files in a minute.

## Cluster setup 
NOTE: prior to deploying you will need to make sure to create the license secret and dockerhub credentials.

For this we are going to use Kustomize to adjust the deployment because, there is a known issue where the
inject istio-proxy prevents upgrade jobs from every completing and terminating. We need to modify the deployment 
to fix this. We'll use kustomize for it.

Step 1 Generate the deployment yaml
------------------------------------
So the way Kustomize works, we need to get the deployment configurations files as input to the kustomize process...since 
we're using Helm templates, we need to synthesize the values and templates so we can operate on the actual deployment yaml file(s).

# This generates the deployment yaml.
helm template anchore -f /Users/jeremybryan/dev/installs/anchore-enterprise/helm/3.2.1/values.yaml anchore/anchore-engine --namespace=anchore > k8s/base/anchore-deploy.yaml

If you look at the k8s/base/anchore-deploy.yaml file, you'll find it's a concatenation of all the deployment files that would be deployed as part of the Helm 
process BUT they are populated with all values from the values files and the namespace as configured in the helm command.

If we wanted to test, at this point we could perform a 
kubectl apply -f k8s/base/anchore-deploy.yaml -n anchore 


Step 2 Modify the deployment with Kustomize
-------------------------------------------
Now we need to back and revisit the kustomize files.

Let's look at the kustomization.yaml file first, here are the contents:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesJson6902:
- path: istio-prevent.json
  target:
    group: batch
    version: v1
    kind: Job
    name: anchore-engine-upgrade
- path: istio-prevent.json
  target:
    group: batch
    version: v1
    kind: Job
    name: anchore-enterprise-upgrade

 Here we've defined 2 patch actions we want to take. We want to apply the patch contained in the istio-prevent.json file to the
 batch/v1 object with name anchore-engine-upgrade AND similarly we want to apply the same patch to the batch/v1 object with name anchore-enterprise-upgrade.

We could list others here if we wanted but for the purposes of demonstration this will suffice. Now let's look at the patch.

[
        {"op": "add",
         "path": "/spec/template/metadata/annotations",
	     "value":
                {"sidecar.istio.io/inject": "false"}
        }
]

Here we have defined an 'add' action to be performed at the specified path ("/spec/template/metadata/annotations"). At that location
we want to inject the following: "sidecar.istio.io/inject": "false". In this case, we are providing an annotation to prevent the istio proxy 
from being injected into the upgrade Job pods.

Let's look at the before view of the target, note the `spec/template/metadata/annotations` section is empty.

# Source: anchore-engine/templates/engine_upgrade_job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "anchore-engine-upgrade"
  labels:
    app.kubernetes.io/managed-by: "Helm"
    app.kubernetes.io/instance: "anchore"
    app.kubernetes.io/version: 1.0.1
    helm.sh/chart: "anchore-engine-1.15.1"
  annotations:
    "helm.sh/hook": post-upgrade
    "helm.sh/hook-weight": "-5"
spec:
  template:
    metadata:
      name: "anchore-engine-upgrade"
      labels:
        app.kubernetes.io/managed-by: "Helm"
        app.kubernetes.io/instance: "anchore"
        helm.sh/chart: "anchore-engine-1.15.1"
      annotations:

    spec:
      securityContext:

Now we actually want to perform the kustomization. To do this we we need to make use of our kustomization.yaml and istio-prevent.json files from earlier. 
Combining these into the following command will result in an updated deployment configuration.

$ kustomize build k8s/overlays/jsonpatch > new.yaml

This command says...perform a build using the configuration defined in the k8s/overlays/jsonpatch/kustomization.yaml file. If you recall,
the kustomizsation.yaml file containers a couple of references:

bases:
- ../../base

patchesJson6902:
- path: istio-prevent.json

The first refers back to the base directory for the original deployment configuration file and the second
points to the istio-prevent.json file which contains the patches we want to apply.

Let's see what happens after we run this command:

The # Source: anchore-engine/templates/engine_upgrade_job.yaml now looks like the following:

apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: post-upgrade
    helm.sh/hook-weight: "-5"
  labels:
    app.kubernetes.io/instance: anchore
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/version: 1.0.1
    helm.sh/chart: anchore-engine-1.15.1
  name: anchore-engine-upgrade
spec:
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "false"
      labels:
        app.kubernetes.io/instance: anchore
        app.kubernetes.io/managed-by: Helm
        helm.sh/chart: anchore-engine-1.15.1
      name: anchore-engine-upgrade
    spec:


Notice the the `spec/template/metadata/annotations` section, it now container sidecar.istio.io/inject: "false" which is what we wanted.

If we recall we had 2 patches we wanted to apply, the second was to the anchore-enterprise-upgrade configuration.  Let's look at that now:

apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: post-upgrade
    helm.sh/hook-weight: "-3"
  labels:
    app.kubernetes.io/instance: anchore
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/version: 1.0.1
    helm.sh/chart: anchore-engine-1.15.1
  name: anchore-enterprise-upgrade
spec:
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "false"
      labels:

We can see that also has been updated with the istio annotation.

Step 3 Deploy the Updated Configuration
---------------------------------------
Once we have the new.yaml output from the kustomization process, we can deploy.

$ kubectl apply -f new.yaml -n anchore 

I noticed that when Istio is deployed, the upgrade containers error out due to an 'unable to connect' condition. This seems
to be related to the pod initialization time being increased with the introduction of istio.  After a short period, the upgrade 
jobs seem to complete normally and Anchore is up.


Miscellaneous Notes
-------------------

## This allows us to ssh into the istio proxy container within the engine upgrade pod
k exec -it pod/anchore-engine-upgrade-wv7dv -n anchore -c istio-proxy -- /bin/sh

## THIS IS HANDY TO SEE THE END RESULT OF A TEMPLATE AFTER INJECTING VALUES VIA VALUES.YAML FILE
helm template anchore -f /Users/jeremybryan/dev/installs/anchore-enterprise/helm/3.2.1/values.yaml anchore/anchore-engine > anchore-deploy.yaml

# To add/remove the istio-injection namespace label 
  kubectl label namespace anchore istio-injection=enabled
  kubectl label namespace anchore istio-injection-
