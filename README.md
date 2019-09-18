This Repository contains all information to run Istio and Ansible Operator workshop on OCP4.

Istio
-----
To try this workshop, you need an OCP cluter with istio. If you don't have one you can use 
[Code Ready Containers](https://developers.redhat.com/blog/2019/09/05/red-hat-openshift-4-on-your-laptop-introducing-red-hat-codeready-containers/) to setup a local development cluster for you.

To setup Istio on OCP cluster please follow: https://access.redhat.com/documentation/en-us/openshift_container_platform/4.1/html/service_mesh/service-mesh-installation#installing-ossm


At fist we need to go through a brief description of Istio components, to understand it best, please read:
https://access.redhat.com/documentation/en-us/openshift_container_platform/4.1/html/service_mesh/service-mesh-architecture#understanding-service-mesh

### First application ###

For our demo purposes we will use istio BookInfo application. It has multiple components and
nicely demonstrate how istio can be usefull for microservices architecture.

``` bash
oc new-project bookinfo
```

deploy default bookinfo application:

``` bash
oc create -f istio/bookinfo.yaml
```

Then we will export our product page url via OpenShift route to not 
need to deploy multiple istio ingress gateways (we can't easily use one
as it allows wildmatching and it will be easy to steal traffic betweeen
workshop participants).

``` bash
oc expose svc productpage
```

Now you can grab URL of your bookinfo application and try to access it via
it's route.

``` bash
oc get route
```

### Destionation Rules ###
To Be able to start using istio features we need to define destination rules
for our application. We can do it easily by applying destination-rule-all.yaml
file to cluster.

``` bash
oc apply -f istio/destination-rule-all.yaml
```

DestinationRules will allow us to play with a routing and other service mesh features.

But first, we will visit our application multiple times. And As you can see we
are getting weird experience, it seems like web page is rendered in a different way each time we access it. 

Lets try kiali to see whats happening.


Tasks
* Explain why we see different results
* Suggest a solution


### Simple routing ###

We will fix our application by routing traffic only to specific service. 
And we will do it by reconfiguring our service mesh. This is very important
that this doesn't require us to change any application code.


``` bash
oc apply -f istio/virtual-service-reviews-v3.yaml
```

``` bash
oc delete -f istio/virtual-service-reviews-v3.yaml
oc apply -f istio/virtual-service-reviews-v2-v3.yaml

```

Tasks
* Use Kiali to route all traffic to reviews-v3 again

### User Specific rules ###

Now we will deploy specific routing for a specific user called redhat.

``` bash

oc apply -f istio/virtual-service-reviews-redhat-v2.yaml

```

Tasks
* Implement route to test user

### Faults ####

Now we will try to break our application by injecting a fault. this will make detail service return error everytime.

``` bash

oc apply -f istio/fault-injection-details-v1.yaml

```

### Delay ###

At the end we will play with delay for review services. We will inject multiple delays and use kialli and Jaeger to debug what happening in the cluster.


``` bash

oc apply -f istio/delay-injection-review-v1.yaml

```


# Create operator which deploys nextcloud

We are in startup mentality here so we decided to go with nextcloud image from DockerHub. In real
life scenario we have to be much more curious and we should setup a security GW to check the 3rd party
images if they're good enough to be used in our environment.

Now its time to jump into operator waters. We will generate our NextCloud Ansible operator. To able to do it
we need following tools installed on your linux machine.

* operator-sdk
* oc client
* ansible
* ansible-runner
* ansible-runner-http

you can use following script to prepate machine.

``` bash
pip install ansible ansible-runner ansible-runner-http openshift
# download and setup operator-sdk
wget https://github.com/operator-framework/operator-sdk/releases/download/v0.10.0/operator-sdk-v0.10.0-x86_64-linux-gnu
chmod +x operator-sdk-v0.10.0-x86_64-linux-gnu 
cp operator-sdk-v0.10.0-x86_64-linux-gnu /usr/local/bin/operator-sdk

# dowload and setup OpenShift client
wget https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz
tar xf openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz 
cp openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit/oc /usr/local/bin
```

We will first need to login to cluster with our account and create project we will run our operator in. If you have runnig OCP 4.x
you can use it. If you don't have one you can use [Code Ready Containers](https://developers.redhat.com/blog/2019/09/05/red-hat-openshift-4-on-your-laptop-introducing-red-hat-codeready-containers/) to setup a local development cluster for you.

``` bash
oc login ...
oc new-project operator
```

Then we will create a new operator and its directory structure by executing:

``` bash
operator-sdk new nextcloud \
  --type ansible --kind Nextcloud \
  --api-version cz.prgcont/v1alpha1
```

This should end up with a directory structure like:

``` bash
nextcloud/
├── build
│   ├── Dockerfile
│   └── test-framework
│       ├── ansible-test.sh
│       └── Dockerfile
├── deploy
│   ├── crds
│   │   ├── cz_v1alpha1_nextcloud_crd.yaml
│   │   └── cz_v1alpha1_nextcloud_cr.yaml
│   ├── operator.yaml
│   ├── role_binding.yaml
│   ├── role.yaml
│   └── service_account.yaml
├── molecule
│   ├── default
│   │   ├── asserts.yml
│   │   ├── molecule.yml
│   │   ├── playbook.yml
│   │   └── prepare.yml
│   ├── test-cluster
│   │   ├── molecule.yml
│   │   └── playbook.yml
│   └── test-local
│       ├── molecule.yml
│       ├── playbook.yml
│       └── prepare.yml
├── roles
│   └── nextcloud
│       ├── defaults
│       │   └── main.yml
│       ├── files
│       ├── handlers
│       │   └── main.yml
│       ├── meta
│       │   └── main.yml
│       ├── README.md
│       ├── tasks
│       │   └── main.yml
│       ├── templates
│       └── vars
│           └── main.yml
└── watches.yaml
```

The most important file for us is *watches.yaml*. It might look like this:

``` yaml
---
- version: v1alpha1
  group: cz.prgcont
  kind: Nextcloud
  role: /opt/ansible/roles/Nextcloud
```

It defines mapping between CR and Ansible Role. This means that every time there is a change in a watched Kubernetes object the Ansible role specified in *watches.yaml* file will be executed.

*Note:* As you see your role is executed frequently so it really must be [Idempotent](https://ryaneschinger.com/blog/ensuring-command-module-task-is-repeatable-with-ansible/)

For more real world style operators please also read about [Finalizers](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/dev/finalizers.md) which enables you to clear all resources which are not stored in Kubernetes.

## Test our operator skelet

To start our development cycle we need to add *CRD* into our Kubernetes cluster. It can 
be achieved by executing following command:

``` bash
oc create -f deploy/crds/cz_v1alpha1_nextcloud_crd.yaml
```

It should end with error telling you, that you are unable to execute this. This is caused by CRD
being cluster scope. I'll precreate this as admin for you, so you can continue. I
ll also introduce a cluster role which will enable you to define custom resoure for this
Custom Resrouce Definition.

Then we will introduce *CR* representing our up into cluster by executing:

``` bash
oc create -f deploy/crds/cz_v1alpha1_nextcloud_cr.yaml
```

And finaly we can execute our operator in test only mode.

``` bash
operator-sdk up local
```

*Note*: To fix an error thrown you need to change your *watches.yaml*
file to contain absolute path to real location of your nextcloud role instead of
*/opt/ansible/roles/nextcloud* so you can run it locally via `operator-sdk up local` command.

It should output at least following:

``` bash
INFO[0000] Running the operator locally.
INFO[0000] Using namespace default.
{"level":"info","ts":1555232629.9483163,"logger":"cmd","msg":"Go Version: go1.11.6"}
{"level":"info","ts":1555232629.9483538,"logger":"cmd","msg":"Go OS/Arch: linux/amd64"}
{"level":"info","ts":1555232629.9483721,"logger":"cmd","msg":"Version of operator-sdk: v0.7.0+git"}
{"level":"info","ts":1555232629.9484124,"logger":"cmd","msg":"Watching namespace.","Namespace":"default"}
{"level":"info","ts":1555232630.6369336,"logger":"leader","msg":"Trying to become the leader."}
{"level":"info","ts":1555232630.636985,"logger":"leader","msg":"Skipping leader election; not running in a cluster."}
{"level":"info","ts":1555232630.6375105,"logger":"proxy","msg":"Starting to serve","Address":"127.0.0.1:8888"}
{"level":"info","ts":1555232630.6377282,"logger":"manager","msg":"Using default value for workers 1"}
{"level":"info","ts":1555232630.637751,"logger":"ansible-controller","msg":"Watching resource","Options.Group":"cz.prgcont","Options.Version":"v1alpha1","Options.Kind":"Nextcloud"}
{"level":"info","ts":1555232630.637985,"logger":"kubebuilder.controller","msg":"Starting EventSource","controller":"nextcloud-controller","source":"kind source: cz.prgcont/v1alpha1, Kind=Nextcloud"}
{"level":"info","ts":1555232630.7382188,"logger":"kubebuilder.controller","msg":"Starting Controller","controller":"nextcloud-controller"}
{"level":"info","ts":1555232630.838377,"logger":"kubebuilder.controller","msg":"Starting workers","controller":"nextcloud-controller","worker count":1}
{"level":"info","ts":1555232632.490276,"logger":"logging_event_handler","msg":"[playbook task]","name":"example-nextcloud","namespace":"default","gvk":"cz.prgcont/v1alpha1, Kind=Nextcloud","event_type":"playbook_on_task_start","job":"8484198340928267159","EventData.Name":"Gathering Facts"}
{"level":"info","ts":1555232633.5219705,"logger":"runner","msg":"Ansible-runner exited successfully","job":"8484198340928267159","name":"example-nextcloud","namespace":"default"}
```

Terminate it with *C-c* and we can continue with updating our operator to be able to deploy
NextCloud instance.

** Deploying nextcloud via Ansible Operator

For our very first deployment will just deploy NextCloud image in default configuration.
In this way there is no external database and everything is stored in internal sqliteDB. For 
the workshop purposes we will not put it on PV.

To be able to deploy our Nextcloud instance on Kubernetes we need to define a three types of objects:

* Deployment
* Service
* Route

To define deployment we *create* a j2 template in: "roles/nextcloud/templates/deployment.yaml.j2"
to contain:

``` jinja
deployment.yaml.j2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ meta.name }}
  namespace: {{ meta.namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ meta.name }}
  template:
    metadata:
      labels:
        app: {{ meta.name }}
    spec:
      containers:
      - image: nextcloud
        name: nextcloud
        ports:
        - containerPort: 80
```

New file a j2 service template will be created in "roles/nextcloud/templates/service.yaml.j2" and contains:

``` jinja
apiVersion: v1
kind: Service
metadata:
  name: {{ meta.name }}
  namespace: {{ meta.namespace }}
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: {{ meta.name }}
```

And last an ingress j2 template will be created as "roles/nextcloud/templates/ingress.yaml.j2" containing:

``` jinja
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ meta.name }}
  namespace: {{ meta.namespace }}
spec:
  port:
    targetPort: 80
  to:
    kind: Service
    name: {{ meta.name }}
    weight: 100
```

At the and we need to add tasks to our Ansible Nexctloud role which will use this templates to define required Kubernetes objects:
*nextcloud/roles/nextcloud/tasks/main.yml*

``` yaml
- name: 'Deploy Nextcloud Instance'
  k8s:
    state: present
    definition: "{{ lookup('template', item.name) | from_yaml }}"
  when: item.api_exists | default(True)
  loop:
    - name: deployment.yaml.j2
    - name: service.yaml.j2
    - name: ingress.yaml.j2
```

And now we run our operator again:

``` bash
operator-sdk up local
```

Now it's time to check our Kubernetes cluster. You should get output similar to this:

``` bash
$ oc get all
NAME                                     READY   STATUS    RESTARTS   AGE
pod/example-nextcloud-58f6679f59-84pkf   1/1     Running   0          170m

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/example-nextcloud   ClusterIP   10.245.170.56    <none>        80/TCP    165m

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example-nextcloud   1/1     1            1           170m

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/example-nextcloud-58f6679f59   1         1         1       170m
```

## Update operator to create a default user

As you probably noted we are unable to access the app directly and we need to set it up a little.
We should adjust our deployment.yaml.j2 template so it creates default user and
we can really log in.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ meta.name }}
  namespace: {{ meta.namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ meta.name }}
  template:
    metadata:
      labels:
        app: {{ meta.name }}
    spec:
      containers:
      - image: nextcloud
        name: nextcloud
        ports:
        - containerPort: 80
        env:
        - name: SQLITE_DATABASE
          value: nextcloud
        - name: NEXTCLOUD_ADMIN_USER
          value: admin
        - name: NEXTCLOUD_ADMIN_PASSWORD
          value: P4ssw0rd
        - name: NEXTCLOUD_TRUSTED_DOMAINS
          value: example-nextcloud-oper.apps.cluster-warsaw-1a90.warsaw-1a90.open.redhat.com
```

And now as you were able to successfuly run your operator localy its time to package it as an docker image
and run it in the openshift cluster properly. Use operator framework to create a docker image with operator 
for you and explain how it should be run in OpenShift cluster.

## K8S Ansible module

Find a documentation to ansible module and update our role to findout URL in the route and set
NEXTCLOUD_TRUSTED_DOMAIN automaticaly.
