------------------------------------------------------------------------------------
                                            # MANAGING DEPLOYMENTS USING GKE:
------------------------------------------------------------------------------------

•	gcloud auth list
•	gcloud config list project
    gcloud config set project [PROJECT_ID]

# SET THE ZONES (Set your working Google Cloud zone by running the following command, substituting the local zone as us-east1-c): 
        gcloud config set compute/zone us-east1-c

1.	Get the sample code for creating and running containers and deployments:
    	gsutil -m cp -r gs://spls/gsp053/orchestrate-with-kubernetes .
    	cd orchestrate-with-kubernetes/kubernetes


2.	Create a cluster with 3 nodes (this will take a few minutes to complete):
	gcloud container clusters create bootcamp \
  --machine-type e2-small \
  --num-nodes 3 \
  --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"

--------------------------------------------
# Task 1. Learn about the deployment object
--------------------------------------------

1.	The explain command in kubectl can tell us about the deployment object:
    •	kubectl explain deployment

2.	You can also see all of the fields using the --recursive option:
    •	kubectl explain deployment --recursive

3.	You can use the explain command as you go through the lab to help you understand the structure of a deployment object and understand what the individual fields do:
    •	kubectl explain deployment.metadata.name

------------------------------
# Task 2. Create a deployment
--------------------------------

1. Update the deployments/auth.yaml configuration file:
    vi deployments/auth.yaml

2. Change the image in the containers section of the deployment to the following:
    ...
    containers:
    - name: auth
      image: "kelseyhightower/auth:1.0.0"
    ...

3. Examine the deployment configuration file:
    cat deployments/auth.yaml

4. Go ahead and create your deployment object using kubectl create:
    kubectl create -f deployments/auth.yaml

5. Once you have created the deployment, you can verify that it was created:
    kubectl get deployments

6. Once the deployment is created, Kubernetes will create a ReplicaSet for the deployment. You can verify that a ReplicaSet was created for the deployment:
    kubectl get replicasets

7. View the Pods that were created as part of the deployment. The single Pod is created by the Kubernetes when the ReplicaSet is created:
    kubectl get pods

8. Use the kubectl create command to create the auth service:
    kubectl create -f services/auth.yaml

9. Now, do the same thing to create and expose the hello deployment:
    kubectl create -f deployments/hello.yaml
    kubectl create -f services/hello.yaml

10. And one more time to create and expose the frontend deployment:
    kubectl create secret generic tls-certs --from-file tls/
    kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
    kubectl create -f deployments/frontend.yaml
    kubectl create -f services/frontend.yaml

11. Interact with the frontend by grabbing its external IP and then curling to it:
    kubectl get services frontend
    curl -ks https://<EXTERNAL-IP>
    OR
    curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`

# Scale a deployment

1. You can look at an explanation of this field using the kubectl explain command again:
    kubectl explain deployment.spec.replicas

2. The replicas field can be most easily updated using the kubectl scale command:
    kubectl scale deployment hello --replicas=5

3. Verify that there are now 5 hello Pods running:
    kubectl get pods | grep hello- | wc -l
    kubectl get pods | grep hello- 

4. Now scale back the application:
    kubectl scale deployment hello --replicas=3

5. Again, verify that you have the correct number of Pods:
    kubectl get pods | grep hello- | wc -l
    kubectl get pods | grep hello- 

-------------------------
# Task 3. Rolling update
-------------------------

1. To update your deployment, run the following command:
    kubectl edit deployment hello

2. Change the image in the containers section of the deployment to the following:
    ...
    containers:
      image: kelseyhightower/hello:2.0.0
    ...

3. See the new ReplicaSet that Kubernetes creates.:
    kubectl get replicaset

4. You can also see a new entry in the rollout history:
    kubectl rollout history deployment/hello

*Pause a rolling update: If you detect problems with a running rollout, pause it to stop the update.
    kubectl rollout pause deployment/hello
    kubectl rollout status deployment/hello

        You can also verify this on the Pods directly:
        kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
    
    kubectl rollout resume deployment/hello
    kubectl rollout resume deployment/hello

# Rollback an update: Assume that a bug was detected in your new version. Since the new version is presumed to have problems, any users connected to the new Pods will experience those issues.

    kubectl rollout undo deployment/hello
    kubectl rollout history deployment/hello
    kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'


---------------------------------
# Task 4. Canary deployments
----------------------------------
A canary deployment consists of a separate deployment with your new version and a service that targets both your normal, stable deployment as well as your canary deployment.

1. First, create a new canary deployment for the new version:
    cat deployments/hello-canary.yaml

2. Now create the canary deployment:
    kubectl create -f deployments/hello-canary.yaml

3. After the canary deployment is created, you should have two deployments, hello and hello-canary. Verify it with this kubectl command:
    kubectl get deployments
    On the hello service, the selector uses the app:hello selector which will match pods in both the prod deployment and canary deployment. However, because the canary deployment has a fewer number of pods, it will be visible to fewer users.

# Verify the canary deployment
    curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version

Canary deployments in production - session affinity
In this lab, each request sent to the Nginx service had a chance to be served by the canary deployment. But what if you wanted to ensure that a user didn't get served by the Canary deployment? A use case could be that the UI for an application changed, and you don't want to confuse the user. In a case like this, you want the user to "stick" to one deployment or the other.

You can do this by creating a service with session affinity. This way the same user will always be served from the same version. In the example below the service is the same as before, but a new sessionAffinity field has been added, and set to ClientIP. All clients with the same IP address will have their requests sent to the same version of the hello application.

Due to it being difficult to set up an environment to test this, you don't need to here, but you may want to use sessionAffinity for canary deployments in production.

------------------------------------
# Task 5. Blue-green deployments
------------------------------------

Rolling updates are ideal because they allow you to deploy an application slowly with minimal overhead, minimal performance impact, and minimal downtime. There are instances where it is beneficial to modify the load balancers to point to that new version only after it has been fully deployed. In this case, blue-green deployments are the way to go.

Kubernetes achieves this by creating two separate deployments; one for the old "blue" version and one for the new "green" version. Use your existing hello deployment for the "blue" version. The deployments will be accessed via a Service which will act as the router. Once the new "green" version is up and running, you'll switch over to using that version by updating the Service.

Note: A major downside of blue-green deployments is that you will need to have at least 2x the resources in your cluster necessary to host your application. Make sure you have enough resources in your cluster before deploying both versions of the application at once

The service:
    Use the existing hello service, but update it so that it has a selector app:hello, version: 1.0.0. The selector will match the existing "blue" deployment. But it will not match the "green" deployment because it will use a different version.

1. First update the service:
    kubectl apply -f services/hello-blue.yaml

# Updating using Blue-Green deployment: In order to support a blue-green deployment style, you will create a new "green" deployment for the new version. The green deployment updates the version label and the image path.

1. Create the green deployment:
    kubectl create -f deployments/hello-green.yaml

2. Once you have a green deployment and it has started up properly, verify that the current version of 1.0.0 is still being used:
    curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version

3. Now, update the service to point to the new version:
    kubectl apply -f services/hello-green.yaml

4. When the service is updated, the "green" deployment will be used immediately. You can now verify that the new version is always being used:
    curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version

# Blue-Green rollback

1. While the "blue" deployment is still running, just update the service back to the old version:
    kubectl apply -f services/hello-blue.yaml

2. Once you have updated the service, your rollback will have been successful. Again, verify that the right version is now being used:
    curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version


========================================================================================================================================
                                                        Congratulations!
This concludes this hands-on lab practicing deployment management with Kubernetes. In this lab you've had the opportunity to work more with the kubectl command-line tool, and many styles of deployment configurations set up in YAML files to launch, update, and scale your deployments. With this foundation of practice you should feel comfortable applying these skills to your own DevOps practice.
========================================================================================================================================




# Orchestrating the Cloud with Kuberenetes

In this Codelab you will learn how to:

* Provision a complete Kubernetes using [Google Container Engine](https://cloud.google.com/container-engine)
* Deploy and manage Docker containers using kubectl

Kubernetes Version: 1.2.2

## Setup GCE and Enable Cloud Shell 

In this section you will create a Google Compute Engine (GCE) account. GCE will allow you to the create VMs, Networks, and Storage volumes required for this workshop. GCE also provides the [Cloud Shell](https://cloud.google.com/shell/docs) computing environment that will be used complete the labs.

#### Labs

  * [Create a GCE Account](labs/create-gce-account.md)
  * [Enable and explore Cloud Shell](labs/enable-and-explore-cloud-shell.md)

### Clone this Repository

Login into your Cloud Shell environment and clone this repository.

```
git clone https://github.com/googlecodelabs/orchestrate-with-kubernetes.git
```

## Provision Kubernetes using GKE

Kubernetes is a distributed system composed of a collection of microservices. Like any system Kubernetes must be installed and configured. In this section you will install Kubernetes from the ground up with the minimal configuration required to get a cluster up and running.

Kubernetes can be configured with many options and add-ons, but can be time consuming to bootstrap from the ground up. In this section you will bootstrap Kubernetes using [Google Container Engine](https://cloud.google.com/container-engine) (GKE).

  * [Provision a Kubernetes Cluster with GKE](labs/provision-kubernetes-cluster-with-gke.md)

## Managing Applications with Kubernetes

Kubernetes is all about applications and in this section you will utilize the Kubernetes API to deploy, manage, and upgrade applications. In this part of the workshop you will use an example application called "app" to complete the labs.

[App](https://github.com/kelseyhightower/app) is hosted on GitHub and provides an example 12 Facter application. During this workshop you will be working with the following Docker images:

* [kelseyhightower/monolith](https://hub.docker.com/r/kelseyhightower/monolith) - Monolith includes auth and hello services.
* [kelseyhightower/auth](https://hub.docker.com/r/kelseyhightower/auth) - Auth microservice. Generates JWT tokens for authenticated users.
* [kelseyhightower/hello](https://hub.docker.com/r/kelseyhightower/hello) - Hello microservice. Greets authenticated users.
* [ngnix](https://hub.docker.com/_/nginx) - Frontend to the auth and hello services.

#### Labs

For each of the following labs, you should be in the kubernetes dir:
```
cd orchestrate-with-kubernetes/kubernetes
```

  * [Creating and managing pods](labs/creating-and-managing-pods.md)
  * [Monitoring and health checks](labs/monitoring-and-health-checks.md)
  * [Managing application configurations and secrets](labs/managing-application-configurations-and-secrets.md)
  * [Creating and managing services](labs/creating-and-managing-services.md)
  * [Creating and managing deployments](labs/creating-and-managing-deployments.md)
  * [Rolling out updates](labs/rolling-out-updates.md)

## Links

  * [Kubernetes](http://googlecloudplatform.github.io/kubernetes)
  * [gcloud Tool Guide](https://cloud.google.com/sdk/gcloud)
  * [Docker](https://docs.docker.com)
  * [etcd](https://coreos.com/docs/distributed-configuration/getting-started-with-etcd)
  * [nginx](http://nginx.org)