
Istio
-----

You can access OpenShift with preinstalled istio via:

If you want to install Istio on your cluster, please follow:

We have access to following components:

*Kiali*: 

It is used to monitor our application traffic and to get
some insight how our application behaves.


*Jaeger*:

It is used to monitor stacktraces of microservices transactions.


### First application ###

For our demo purposes we will use istio BookInfo application. It has multiple components and
nicely demonstrate how istio can be usefull for microservices architecture.

``` bash
export project="REPLACE-ME"

oc new-project $project

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
