# OpenShift Routes verse Custom Resource Definitions

This document compares the differences between OpenShift Routes and Custom Resource Definitions. Custom Resource allows you to extend Kubernetes capabilities by adding any kind of API object useful for your application. Custom Resource Definition is what you use to define a Custom Resource. This is a powerful way to extend F5 CIS Kubernetes capabilities beyond the default installation and similar to OpenShift Routes.

In this example we are using a cafe application with three endpoints; **tea,coffee and mocha** as shown in the diagram below

![architecture](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-26_14-39-29.png)

Demo on YouTube [video]()

## Using OpenShift Route

### Step 1: Deploy CIS

Currently in CIS 2.7 only one Public IP **Virtual Server** for BIG-IP can be configured for all Routes. Routes uses HOST Header Load balancing to determine the backend
application. In this example the backend is **/tea,/coffee and /mocha**

Add the following parameters to the CIS deployment

* --route-vserver-addr=10.192.125.65 - Public IP for BIG-IP for all Routes
* --manage-routes=true - Configure CIS to watch for Routes
* --bigip-partition=OpenShift - CIS uses BIG-IP tenant OpenShift to manage Routes
* --openshift-sdn-name=/Common/openshift_vxlan - CNI policy on BIG-IP to connect to the PODs in OpenShift
* --override-as3-declaration=default/cafe-override - AS3 Override allows you to add Objects not exposed by an annotations such as custom policies, iRules, profiles etc

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
  "--bigip-username=$(BIGIP_USERNAME)",
  "--bigip-password=$(BIGIP_PASSWORD)",
  "--bigip-url=10.192.125.60",
  "--bigip-partition=OpenShift",
  "--namespace=default",
  "--pool-member-type=cluster",
  "--openshift-sdn-name=/Common/openshift_vxlan",
  "--insecure=true",
  "--manage-routes=true",
  "--route-vserver-addr=10.192.125.65",
  "--override-as3-declaration=default/cafe-override",
  "--as3-validation=true",
  "--log-as3-response=true",
]
```

Deploy CIS in OpenShift

```
oc create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
oc create -f bigip-ctlr-clusterrole.yaml
oc create -f f5-bigip-ctlr-deployment.yaml
```

CIS [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/route-vs-crd/route/cis)

**Note** Do not forget the CNI [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/route-vs-crd/route/cni)

### Step 2: Creating OpenShift Routes

User-case for the OpenShift Routes:

- Edge Termination
- Redirect HTTP to HTTPS
- Health monitor of the backend NGINX application using HOST **cafe.example.com** and **PATH /coffee, /tea and /mocha**
- Custom HTTP Policy for X-Forwarded-For (XFF) HTTP header field - **Use AS3 override**
- Backend listening on PORT 8080