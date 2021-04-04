# OpenShift Networking Demo

This repository helps to demonstrate some basic features of OpenShift/Kubernetes networking, such as:

* Ingress using the OpenShift Router (HA Proxy), including edge terminated TLS.
* NetworkPolicy to restrict traffic from entering or leaving a project/namespace.
* NetworkPolicy to allow Ingress from the Router into the project/namespace.
* NetworkPolicy to allow fine-grained traffic management between projects/namespaces.

## Setup

Run the following commands to create the Pet Clinic project and Pet Clinic application, as well as a MySQL database in it's own database project.

```
oc apply -k resources/app-environment

oc apply -k resources/db-environment
```

This will create the initial infrastructure.  

If you look at the Pet Clinic *Deployment* you will see it is attempting to connect to the database at: `petclinicdb.networkdemo-database.svc.cluster.local` 

This is using the internal cluster SDN.  This is an internal DNS address that is derived as follows:

`<service name>.<namespace>.svc.cluster.local`

So for the URL referenced by the Pet Clinic application, it is looking for a service named `petclinicdb` in the `networkdemo-database` namespace.

The application will fail to start because it can't reach the database due to the *Deny All* policy.  Ingress to the application through the router will also fail because the Pet Clinic project has been configured to *Deny All* network traffic from outside of it's project.  This includes the router.

## Examining the Pet Clinic and Databases NetworkPolicy

Both of these projects (*networkpolicy-petclinic* and *networkpolicy-database*) have the same default `NetworkPolicy` applied.  This allows traffic between pods in the same namespace, but denies traffic from other namespaces and the router:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-same-namespace
spec:
  podSelector:
  ingress:
  - from:
    - podSelector: {}
```

This has the following effect on the projects:

* No ingress (even from the router) is allowed to applications in these projects.
* The Pet Clinic application will not be allowed to communicate with the MySQL database, since the *networkdemo-database* namespace is denying all incoming traffic.

## Fixing the Database Connection

First, let's fix the database connection.

`NetworkPolicy` objects are additive, so there is no reason to remove or edit the existing *deny all* policy.  Instead, we will add another `NetworkPolicy` object that will allow the minimum amount of traffic required.  In this case, the only traffice we want into this project is:
* Only allow access from the *networkdemo-petclinic* project, and
* Only on port 3306 (MySQL), and
* Only to the pod with the `name=petclinicdb` label (MySQL)

This looks like:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-petclinic-to-mysql
  namespace: networkdemo-database
spec:
  podSelector:
    matchLabels:
      name: petclinicdb
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              app: petclinic
      ports:
        - protocol: TCP
          port: 3306
```

You can apply it to the *networkpolicy-database* project by running:

```
 oc apply -f resources/networkpolicies/allow-petclinic-to-mysql.yaml -n networkpolicy-database
```

You will now see both `NetworkPolicy` objects in the *networkdemo-database* project.  If you check the logs for the Pet Clinic pod, you should see that it has now connected to the database!  If not, you might need to kill the pod so that it starts up again and connects to the database.

## Fixing Ingress Through the Router

Now that the application is healthy and connects to the database, we need to be able to connect to the Pet Clinic application from a web browser. To do this, we need to allow ingress to the *networkdemo-petclinic* project from the Router.

This `NetworkPolicy` looks like this:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress
  podSelector: {}
  policyTypes:
  - Ingress
```

This will allow ingress from the namespace where the OpenShift router (HA Proxy) resides.

Apply this `NetworkPolicy` to the *networkdemo-petclinic* namespace by running:

```
oc apply -f resources/networkpolicies/allow-from-openshift-ingress.yaml -n networkdemo-petclinic
```

The *networkdemo-petclinic* project will now have both the `allow-from-same-namespace` and `allow-from-openshift-ingress` policies applied.

Now, if you try to access the Pet Clinic application through it's generated `Route` URL, you will be able to access the app.

This is now a fully functional app that has a web UI and a database with proper network firewall rules applied with `NetworkPolicy` objects~

## About the OpenShift Router

If your OpenShift cluster is configured with a valid *wildcard* certificate for the `*.apps` domain name, then the OpenShift Router can automatically create *secure* TLS URLS for your applications.  This is accomplished with the `edge` TLS termination for routes.  With this option enabled, a route will have a secure URL where TLS is terminated at the HA Proxy router.  At this point, traffic is unencrypted, but this unencrypted traffic is only *internal* to the cluster on the SDN.

If greater security is required, the OpenShift Router offers two other options for TLS configuration:

* **Passthrough** - TLS traffic is not decrypted, but instead passed directly to the application where the application is responsible for maintaining proper TLS certificates.  This will use TLS SNI (Server Name Indication) to pass encrypted traffic to the pods/applications.

* **Re-Encrypt** - The OpenShift router will terminate TLS, then use a different certificate to re-encrypt traffice before passing it to the backing pod/application.


