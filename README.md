# BlackLabelOps/GCE-Jenkins

Google Container Engine (GCE) is coming and it's time to deploy some Docker containers! This tutorial will show how to deploy an application inside the GCE cloud.

The tutorial will use my Jenkins Docker Container: [blacklabelops/jenkins](https://github.com/blacklabelops/jenkins).

The tutorial will provide some prime examples along with plenty of manifest files in order to show the different possible configurations and extension capabilities.

Learn how to:

* Deploy a Docker Container in GCE.
* Inspect and Proper Removal of Cloud Components
* Accessing Container from the Internet.
* Setup Persistence, EMail and Logging.

This project includes tutorial-like manual for deploying the container and usable
templates in yaml format for deployment.

Required software:

* Vagrant
* Virtualbox

Required Accounts:

* Google Cloud Access

# Prelude

You will have to clone this github project on your local computer. The tutorial needs some files out of this repository, e.g. Vagrantfile, Kubernetes Manifests.

You must have access to a Google Developers Cloud console at

[https://console.developers.google.com/project](https://console.developers.google.com/project)

* Create a new project, e.g.: gce-jenkins
* Remember your project-id, e.g.: gce-jenkins-324

# Configuration

The software installation is already done inside my Vagrant box. My blacklabelops/dockerdev box
has the appropriate Docker and Google Cloud SDK version. Just fire up the box and login with the
following commands:

~~~~
$ vagrant up
$ vagrant ssh
~~~~

> Uses the local Vagrantfile to fire up the Virtualbox image.

Now login to your google cloud account with the Google Cloud SDK (gcloud) command line interface (cli):

~~~~
$ gcloud auth login
Go to the following link in your browser:

    https://accounts.google.com/o/oauth2/...
~~~~

Copy & Paste the link into your browser. Follow the procedure and Copy & Paste the verification code into your console:

~~~~
Enter verification code: ...
Saved Application Default Credentials.

You are now logged in as ...
~~~~

Fixing the Cli to a specific project. Afterwards every command will be interpreted as a command for this specific project:

~~~~
$ gcloud config set project gce-jenkins-324
~~~~

Setup your default zone:

~~~~
$ gcloud config set compute/zone europe-west1-b
~~~~

> Zones are defined [here](https://cloud.google.com/compute/docs/zones). In my case I want my infrastructure in europe.

Finally you need to enable the Google Cloud Compute Engine for your API. I usually do this manually by clicking **Compute/VM instances/** and clicking **Container Engine/Container cluster** inside the Developers Console. There will be a popup saying "Initializing...". Otherwise the cli will break up with "internal error" during this tutorial.

Now your ready to go! Fire up some kubernetes clusters!

# Deployment

It's time to start a cluster with Kubernetes nodes! Each node is in essence a Google Compute Virtual Machine and preconfigured with Kubernetes. In this example I need one node because scaling the jenkins master makes no sense. I will provide an example with scaled slaves later on.

Starting a Kubernetes Cluster:

~~~~
$ gcloud beta container clusters create gce-jenkins-cluster \
  --disk-size 50 \
  --machine-type g1-small \
  --num-nodes 1 \
  --zone europe-west1-b
~~~~

> Note: Zones are defined [here](https://cloud.google.com/compute/docs/zones) and machine-types are defined [here](https://cloud.google.com/compute/docs/machine-types).

You may want to look into your machine but it's usually a just look and do not touch anything because the image is kubernetes managed and should not be modified. First you need to get your machines id and then just ssh into it:

~~~~
$ gcloud compute instances list
NAME                                  ZONE           MACHINE_TYPE PREEMPTIBLE INTERNAL_IP   EXTERNAL_IP   STATUS
gke-jenkins-master-30f1afd0-node-uvl8 europe-west1-b g1-small                 10.240.128.33 104.155.29.68 RUNNING
$ gcloud compute ssh gke-jenkins-master-30f1afd0-node-uvl8
... some commands ...
$ exit
~~~~

> Note: Don't forget to exit the machine when you continue my tutorial!

In Kubernetes slang we haven't deployed a virtual machine but a Cluster. *Cluster : A cluster is a set of physical or virtual machines and other infrastructure resources used by Kubernetes to run your applications.* - [Kubernetes Github](https://github.com/googlecloudplatform/kubernetes).

The following command are executed by kubectl. At this point we leave the gcloud world and enter the world of Kubernetes. It means we have acquired the necessary computing resources and dive in the world of cloud containers.

Next step is running the Jenkins Docker container. In Kubernetes slang we don't deploy a container but we define and deploy a pod. *Pod: Pods are a colocated group of application containers with shared volumes. They're the smallest deployable units that can be created, scheduled, and managed with Kubernetes* - [Kubernetes Github](https://github.com/googlecloudplatform/kubernetes).

Pods are defined by manifest files structured in json or yaml. I prefer yaml for readability. My jenkins pod specification can be found in the file: *gce-jenkins-simple-pod-manifest.yaml*

~~~~
apiVersion: v1
kind: Pod
metadata:
  name: gce-jenkins-pod
  labels:
    name: gce-jenkins-pod
spec:
  containers:
    - name: gce-jenkins
      image: docker.io/blacklabelops/jenkins
      imagePullPolicy: Always
      ports:
        - containerPort: 8080
          targetPort: 8080
  restartPolicy: Always
  dnsPolicy: Default
~~~~

> This file references my image on [Docker Hub](https://registry.hub.docker.com/u/blacklabelops/jenkins/). This is the default startup command of my container.

Let's start the pods and see what's happening.

~~~~
$ kubectl create -f /vagrant/gce-jenkins-pod-simple-manifest.yaml
pods/gce-jenkins-pod
~~~~

> The /vagrant folder is a default mount by Vagrant inside the local project folder. You need to clone this project in order to be able to execute it!

List the pods in  order to see it's status:

~~~~
$ kubectl get pods
NAME              READY     REASON    RESTARTS   AGE
gce-jenkins-pod   1/1       Running   0          1m
~~~~

See the logs in order to check Jenkins:

~~~~
$ kubectl logs gce-jenkins-pod
...
INFO: Jenkins is fully up and running
~~~~

Now let's test Jenkins. Kubernetes can setup port forwarding to any pod. So we can access it locally.

~~~~
$ nohup kubectl port-forward -p gce-jenkins-pod 8080:8080 &
$ curl http://localhost:8080
...
<title>Dashboard [Jenkins]</title>
...
~~~~

> Nohup will run the port-forwarding in the background so we can poll Jenkins with Curl.

# Jenkins Internet Access

In order to be able to access Jenkins we need to expose it as a Kubernetes Service. *Service : Services provide a single, stable name and address for a set of pods. They act as basic load balancers.* - [Kubernetes Github](https://github.com/googlecloudplatform/kubernetes).

The definition of our example service is as follows:

~~~~
apiVersion: v1
kind: Service
metadata:
  name: gce-jenkins-service
  labels:
    name: gce-jenkins-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    name: gce-jenkins-pod
~~~~

> Note the reference on the pod by a *selector*.

Let's create the Service/LoadBalancer for our Jenkins pod. The service is instantiated as follows.:

~~~~
$ kubectl create -f /vagrant/gce-jenkins-service-simple-manifest.yaml
An external load-balanced service was created.  On many platforms (e.g. Google Compute Engine),
			you will also need to explicitly open a Firewall rule for the service port(s) (tcp:80) to serve traffic.
~~~~

As the message says we have finished the Kubernetes part but still have to open the Google Cloud firewall. We need the node prefix for the firewall rule then adding the firewall rule to the global rule list:

~~~~
$ kubectl get nodes
NAME                                         LABELS                                                              STATUS
gke-gce-jenkins-cluster-eb944e83-node-bqbp   kubernetes.io/hostname=gke-gce-jenkins-cluster-eb944e83-node-bqbp   Ready
$ gcloud compute firewall-rules create \
    gce-jenkins-80 \
    --allow tcp:80 \
    --target-tags gke-gce-jenkins-cluster-eb944e83-node
~~~~

> Note: Only take the node prefix! Omit: -bqbp

Now access Jenkins with your browser! Look for the external IP of your Service and open it inside your web browser:

~~~~
$  kubectl describe services gce-jenkins
Name:			gce-jenkins-service
Labels:		name=gce-jenkins-service
Selector:		name=gce-jenkins-pod
Type:			LoadBalancer
IP:			   10.11.242.162
LoadBalancer Ingress:	104.155.16.238
Port:			<unnamed>	80/TCP
NodePort:		<unnamed>	32062/TCP
Endpoints:		10.8.0.8:8080
Session Affinity:	None
No events.
~~~~

> The Ip you need is inside the field *LoadBalancer Ingress*. Example URL: http://104.155.16.238

## Jenkins Setup & Persistence

We have Jenkins up and running and it's time to properly setup Jenkins. Jenkins should have a persistent volume so that we can throw away the container without losing data.

Okay first delete our old pod:

~~~~
$ kubectl delete pod gce-jenkins-pod
~~~~

Create a Google Cloud Disk for our container volume:

~~~~
$ gcloud compute disks create \
  --size 200GB \
  --zone europe-west1-b \
  gce-jenkins-disk
Created [https://www.googleapis.com/compute/v1/projects/gce-jenkins/zones/europe-west1-b/disks/gce-jenkins-disk].
NAME             ZONE           SIZE_GB TYPE        STATUS
gce-jenkins-disk europe-west1-b 200     pd-standard READY
~~~~

> A 200GB disk with Id gce-jenkins-disk. Check this on how to manage Google Disks: [Documentation](https://cloud.google.com/compute/docs/disks/persistent-disks).

My configuration for Jenkins:

* Increasing Java Heap Space -Xmx1024m -Xms256m
* Default Security Setup:
    * Admin-User "jenkins"
    * Password "swordfish"
* Jenkins Pod with default plugins: git hipchat swarm
* Enable slave connection on fixed port 50000.

The pod configuration (gce-jenkins-pod-setup-manifest.yaml) now looks like this:

~~~~
apiVersion: v1
kind: Pod
metadata:
  name: gce-jenkins-pod
  labels:
    name: gce-jenkins-pod
spec:
  containers:
    - name: gce-jenkins
      image: docker.io/blacklabelops/jenkins
      imagePullPolicy: Always
      ports:
        - containerPort: 8080
          targetPort: 8080
        - containerPort: 50000
          targetPort: 50000
      env:
        - name: JENKINS_ADMIN_USER
          value: jenkins
        - name: JENKINS_ADMIN_PASSWORD
          value: swordfish
        - name: JENKINS_PLUGINS
          value: git hipchat swarm
        - name: JENKINS_SLAVEPORT
          value: "50000"
        - name: JAVA_VM_PARAMETERS
          value: -Xmx1024m -Xms256m
      volumeMounts:
        - name: jenkins-master-persistent-storage
          mountPath: /jenkins
  volumes:
      - name: jenkins-master-persistent-storage
        gcePersistentDisk:
          pdName: gce-jenkins-disk
          fsType: ext4
  restartPolicy: Always
  dnsPolicy: Default
~~~~

Let's fire up our cloud-ready Jenkins master pod!

~~~~
$ kubectl create -f /vagrant/gce-jenkins-pod-setup-manifest.yaml
~~~~

This may now take some time because the disk is being prepared. Check the status regularily:

~~~~
$ kubectl get pod
NAME              READY     REASON                                                                   RESTARTS   AGE
gce-jenkins-pod   0/1       Image: docker.io/blacklabelops/jenkins is ready, container is creating   0          7m
~~~~

# Jenkins HTTPS

It's also possible to deploy HTTPS Jenkins inside the Google Container Engine. HTTPS uses a different port, therefore we have to clean up and restart the tutorial from zero.

> Note: HTTPS-Functionality is executed by Jenkins itself. There is also a HTTPS-LoadBalancer by Google but currently not in operation.

My additional configuration for Jenkins:

* Password for Jenkins certificate keystore: keystoreswordfish
* Distinguished Name (DN) for SSL certificate: CN=JenkinsTutorial,OU=Blacklabelops,O=blacklabelops.net,L=Munich,S=Bavaria,C=DE

New manifest for Jenkins HTTPS Pod (gce-jenkins-pod-https-manifest.yaml):

~~~~
apiVersion: v1
kind: Pod
metadata:
  name: gce-jenkins-pod
  labels:
    name: gce-jenkins-pod
spec:
  containers:
    - name: gce-jenkins
      image: docker.io/blacklabelops/jenkins:latest
      imagePullPolicy: Always
      ports:
        - containerPort: 8080
          name: jenkins-https
        - containerPort: 50000
          name: jenkins-slave
      env:
        - name: JENKINS_ADMIN_USER
          value: jenkins
        - name: JENKINS_ADMIN_PASSWORD
          value: swordfish
        - name: JENKINS_PLUGINS
          value: git hipchat swarm
        - name: JENKINS_SLAVEPORT
          value: "50000"
        - name: JAVA_VM_PARAMETERS
          value: -Xmx1024m -Xms256m
        - name: JENKINS_KEYSTORE_PASSWORD
          value: keystoreswordfish
        - name: JENKINS_CERTIFICATE_DNAME
          value: CN=JenkinsTutorial,OU=Blacklabelops,O=blacklabelops.net,L=Munich,S=Bavaria,C=DE
      volumeMounts:
        - name: jenkins-master-persistent-storage
          mountPath: /jenkins
  volumes:
      - name: jenkins-master-persistent-storage
        gcePersistentDisk:
          pdName: gce-jenkins-disk
          fsType: ext4
  restartPolicy: Always
  dnsPolicy: Default
~~~~

New manifest for HTTPS Service/Loadbalancer (gce-jenkins-service-https-manifest.yaml):

~~~~
apiVersion: v1
kind: Service
metadata:
  name: gce-jenkins-service
  labels:
    name: gce-jenkins-service
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: jenkins-https
      protocol: TCP
      name: https
    - port: 50000
      targetPort: jenkins-slave
      protocol: TCP
      name: slave-port
  selector:
    name: gce-jenkins-pod
~~~~

Starting the Kubernetes Cluster:

~~~~
$ gcloud beta container clusters create gce-jenkins-cluster \
  --disk-size 50 \
  --machine-type g1-small \
  --num-nodes 1 \
  --zone europe-west1-b
~~~~

Create a Google Cloud Disk for our container volume:

~~~~
$ gcloud compute disks create \
  --size 200GB \
  --zone europe-west1-b \
  gce-jenkins-disk
Created [https://www.googleapis.com/compute/v1/projects/gce-jenkins/zones/europe-west1-b/disks/gce-jenkins-disk].
NAME             ZONE           SIZE_GB TYPE        STATUS
gce-jenkins-disk europe-west1-b 200     pd-standard READY
~~~~

Starting the Pod:

~~~~
$ kubectl create -f /vagrant/gce-jenkins-pod-https-manifest.yaml
~~~~

Starting the Service:

~~~~
$ kubectl create -f /vagrant/gce-jenkins-service-https-manifest.yaml
An external load-balanced service was created.  On many platforms (e.g. Google Compute Engine),
			you will also need to explicitly open a Firewall rule for the service port(s) (tcp:80) to serve traffic.
~~~~

Open the firewall

~~~~
$ kubectl get nodes
NAME                                         LABELS                                                              STATUS
gke-gce-jenkins-cluster-eb944e83-node-bqbp   kubernetes.io/hostname=gke-gce-jenkins-cluster-eb944e83-node-bqbp   Ready
$ gcloud compute firewall-rules create \
    gce-jenkins-443 \
    --allow tcp:443 \
    --target-tags gke-gce-jenkins-cluster-eb944e83-node
~~~~

> Note: We only open the firewall for UI access. Slaves will be started inside the Container Engine.

Now access Jenkins with your browser. Look for the external IP of your Service and open it inside your web browser:

~~~~
$  kubectl describe services gce-jenkins
...
LoadBalancer Ingress:	104.155.16.238
...
Session Affinity:	None
No events.
~~~~

> The Ip you need is inside the field *LoadBalancer Ingress*. Example URL: https://104.155.16.238

## Cleaning Up

Cleaning Up all of the tutorial components!

Delete the service.

~~~~
$ kubectl delete service gce-jenkins-service
~~~~

Delete the pod.

~~~~
$ kubectl delete pod gce-jenkins-pod
~~~~

Delete the cluster

~~~~
$ gcloud beta container clusters delete gce-jenkins-cluster
~~~~

Delete the firewall rule

~~~~
$ gcloud compute firewall-rules delete gce-jenkins-80
~~~~

Delete the disk

~~~~
$ gcloud compute disks delete gce-jenkins-disk
~~~~

Delete the Vagrant box

~~~~
$ vagrant destroy
~~~~

> Note: Exit the VM by typing 'exit'.

## References

* [Kubernetes](http://kubernetes.io/)
* [Kubernetes Github](https://github.com/googlecloudplatform/kubernetes)
* [Google Cloud](https://cloud.google.com/)
* [Google Developers Cloud Console](https://console.developers.google.com/project)
* [Jenkins - Getting Jenkins Cloud Ready - Part 1](http://stbleul.blogspot.de/2015/07/jenkins-getting-jenkins-cloud-ready.html)
