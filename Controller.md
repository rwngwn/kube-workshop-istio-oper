# Kubernetes Scheduling #


The purpose of this workshop is to help you expand your Kubernetes
knowledge. In this part we will look how to run more sophisticated
deployments and will show you how you can use python to easily interact
with Kubernetes API server. We will be using Gordon application created
for workshop.

## Kubernetes API

In this chapter we will start to talk with Kubernetes API. We will do it by writing a very simple pod scheduler.
Language of our choice is Python, but you can use almost any other language - the principles will the same.

### Preparing Python Env

We will start by creating Python virtual environment

``` bash

oc new-app python~https://github.com/GrahamDumpleton/os-sample-python.git --name shell

oc get pods
```

Wait for a pod to be deployed and then get shell to it and install python kubernetes bindings
``` bash
oc exec -ti  shell-1-mhc8g bash
pip install kubernetes
```

### Preparing pod

We will now create a pod which will tell Kubernetes to wait for our custom scheduler.
You can do it by feeding Kubernetes with following YAML:

``` bash
oc apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  schedulerName: PrgContSched
  containers:
  - name: hello
    image: prgcont/gordon:v1.0
EOF
```

Now if we will run *oc get pods* we'll see our pod stuck in a *Pending* state.
This is the initial state of a pod and Kubernetes default scheduler is ignoring this pod as we marked our pod to be schedulable via *PrgContSched* scheduler.

### Talking to Kubernetes API

First we need to grant requested permissions to your service account:

``` bash
oc adm policy add-role-to-user admin system:serviceaccount:$PROJECT:default
```

First we will create a simple Python script which will connect to Kubernetes and will list all Pods waiting to be scheduled:

``` python
from kubernetes import client, config, watch

config.load_incluster_config()
v1 = client.CoreV1Api()
namespace = "david"

def main():
    w = watch.Watch()
    for event in w.stream(v1.list_namespaced_pod, namespace):
        print("pod: '%s', phase: '%s'." % (event['object'].metadata.name,
                                           event['object'].status.phase))

if __name__ == '__main__':
    main()
```

You should see your pod in the =Pending= state.

### Tasks

- Look at [[https://github.com/kubernetes-client/python/blob/master/kubernetes/docs/V1Pod.md][Kubernetes Python API docs]] and adjust python script to print name of the requested scheduler too.

## Scheduling a Pod

To be able to schedule our pod we will create a simple Schedule function:

``` python
def scheduler(name, node, namespace=namespace):
        
    target=client.V1ObjectReference()
    target.kind = "Node"
    target.apiVersion = "v1"
    target.name = node
    
    meta = client.V1ObjectMeta()
    meta.name = name
    
    body = client.V1Binding(metadata=meta, target=target)
    
    return v1.create_namespaced_binding(namespace, body)
```

and adjust our main function to look like:

``` python
def main():
    w = watch.Watch()
    for event in w.stream(v1.list_namespaced_pod, namespace):
        print("pod: '%s', phase: '%s' %s." % (event['object'].metadata.name,
                                           event['object'].status.phase,
                                           event['object'].spec.scheduler_name))
        if event['object'].status.phase == "Pending" and event['object'].spec.scheduler_name == "PrgContSched":
            try:
                res = scheduler(event['object'].metadata.name, 'ip-10-0-131-66.eu-central-1.compute.internal')
                print(res)
            except Exception as ex:
                print(ex)
```

When you run your function it should schedule a pod. If you get an exception you can probably ignore it as there is currently bug in this [[https://github.com/kubernetes-client/gen/issues/52][API]].

Check pod state by invoking:

``` bassh
$ oc get pods
``` 

*** TASK: Moving Your Scheduler Inside of Your Cluster
- Build this scheduler as a Docker image
- Deploy it to the Kubernetes
- Schedule a pod with it