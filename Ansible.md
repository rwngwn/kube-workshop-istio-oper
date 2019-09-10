** Create operator which deploys nextcloud

We are in startup mentality here so we decided to go with nextcloud image from DockerHub. In real 
life scenario we have to be much more curious and we should setup a security GW to check the 3rd party
images if they're good enough to be used in our environment.

Now its time to jump into operator waters. We will generate our NextCloud Ansible operator

``` bash
wget https://github.com/operator-framework/operator-sdk/releases/download/v0.10.0/operator-sdk-v0.10.0-x86_64-linux-gnu
chmod +x operator-sdk-v0.10.0-x86_64-linux-gnu 
./operator-sdk-v0.10.0-x86_64-linux-gnu 
mv operator-sdk-v0.10.0-x86_64-linux-gnu operator-sdk


https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz
tar xf openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz 
alias oc=`pwd`/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit/oc

pip install ansible-runner
pip install ansible

mkdir ansible
cd ansible/
```


directory structure by executing:
``` bash
../operator-sdk new nextcloud \
   --type ansible --kind Nextcloud \
   --api-version cz.prgcont/v1alpha1 
```

This should end up with a directory structure like:
```
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

*Note:* As you see your role is executed frequently so it really must be [[https://ryaneschinger.com/blog/ensuring-command-module-task-is-repeatable-with-ansible/][idempotent]].

For more real world style operators please also read about [[https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/dev/finalizers.md][finalizers]], which enables you to clear
all resources which are not stored in Kubernetes.


** Test our operator skelet
To start our development cycle we need to add *CRD* into our Kubernetes cluster. It can 
be achieved by executing:

``` bash
a
```


Then we will introduce *CR* representing our up into cluster by executing:


``` bash
oc create -f deploy/crds/cz_v1alpha1_nextcloud_cr.yaml
```


And finaly we can execute our operator in test only mode.
``` bash
../../operator-sdk up local
```

*Note*: Currently there is a bug in Ansible operator framework and you need to change your *watches.yaml*
file to contain absolute path to real location of your nextcloud role instead of
*/opt/ansible/roles/nextcloud* so you can run it locally via `operator-sdk up local` command.


It should output at least following:
```
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
- Deployment
- Service
- Ingress

To define deployment we *create* a j2 template in ~nextcloud/roles/nextcloud/templates/demployment.yaml.j2~
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


New file a j2 service template will be created in roles/nextcloud/templates/service.yaml.j2 and contains:

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


And last an ingress j2 template will be created as roles/nextcloud/templates/ingress.yaml.j2 containing:

``` jinja
apiVersion: route.openshift.io/v1
kind: Route
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ meta.name }}
  namespace: {{ meta.namespace }}
spec:
  rules:
  - host: {{ dns }}.apps.prgcont.cz
    port:
      targetPort: 80-tcp
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


After creating files above we will run the operator again
``` bash
operator-sdk up local
```

*Pro-tip:* you can access Ansible logs via 
~/tmp/ansible-operator/runner/cz.prgcont/v1alpha1/Nextcloud/default/example-nextcloud/artifacts/latest/stdout~.

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