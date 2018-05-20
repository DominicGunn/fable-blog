---
author: "Dominic Gunn"
title: "Securing Kubernetes cluster access"
date: 2018-05-19T22:07:16+01:00
categories:
  - "kubernetes"
  - "security"
---

In a growing world it's become the norm for teams to be able to own their services, in their entirety. Everything from initial planning to development and finally deployment. Gone are the days of throwing the code over the wall to an Ops team you never met or only had arguments with, welcome to the future.

At [Turnitin](www.turnitin.com) we're beginning to strongly leverage the power of Kubernetes for our deployments. We have a large breadth of teams working on anything from integrations with LMSs (Moodle, Blackboard, Canvas, etc.), to the powerful services we use for originality matching, all of this has to be housed somewhere and housed in such a fashion that the teams responsible for the services are able to manage their deployments.

## Enter namespaces

We're going to play with [minikube](https://kubernetes.io/docs/getting-started-guides/minikube/) in order to leverage the power of namespaces, let's take a look at what's provided by default.

```bash
➜  ~ kubectl get namespaces
NAME          STATUS    AGE
default       Active    69d
kube-public   Active    69d
kube-system   Active    69d
```

This is great, but our teams are going to need a namespace of their own, we can create namespaces using `kubectl`! Check out this [gist](https://gist.github.com/DominicGunn/6af344f949333cbb90b32b3701833c87) for a description of **team-a**'s namespace. Let's use that to set up an area for the team.

It contains a little bit of information regarding the namespace we're going to create, most importantly the name!

```bash
➜  ~ kubectl create -f team-a-namespace.json
namespace "team-a" created

➜  ~ kubectl get namespaces
NAME          STATUS    AGE
default       Active    69d
kube-public   Active    69d
kube-system   Active    69d
team-a        Active    9s
```

There's a variety of super great policies that can be defined when creating namespaces, one that you might be interested to look at involves [resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/).

## Handling deployments

A namespace for the team is no good if they can't put anything there. We don't want to give every team super access to the cluster so we should create a service account that has permissions to manage their namespace for them.

Kubernetes has a variety of authorization mechanisms but the one we're going to be looking at is [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/), specifically [Role](https://kubernetes.io/docs/admin/authorization/rbac/#role-examples)'s and [RoleBinding](https://kubernetes.io/docs/admin/authorization/rbac/#rolebinding-and-clusterrolebinding)'s.

### <u>Creating the ServiceAccount</u>

Creating a service account is a simple process, in order to do that we need to define the name of the account, and also define the namespace in which it will live, the gist is [here](https://gist.github.com/DominicGunn/ca57b205930039224d05bb227d5ef3fc), but it's small enough that we can take a look.

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-a-service-account
  namespace: team-a
```

Super easy, let's have `kubectl` create that for us.

```bash
➜  ~ kubectl create -f team-a-service-account.yaml
serviceaccount "team-a-service-account" created
```

### <u>Creating the Role</u>

We have a service account, but it can't do anything yet, we need to give it a role. The [documentation](https://kubernetes.io/docs/admin/authorization/rbac/#role-examples) is super great, so I suggest you take a read, for the sake of this team a service account that permissions create and update deployments and services will fit the teams needs.

The definition for such a role is available on gist [here](https://gist.github.com/DominicGunn/ca57b205930039224d05bb227d5ef3fc) and looks like this

```bash
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: team-a
  name: team-a-deploy-role
rules:
- apiGroups: ["extensions", "apps"]
  resources: ["deployments"]
  verbs: ["get", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "create", "update", "patch"]
```

Again not much to it, we have defined a couple of explicit rules for the role that perfectly fit what our team might need. We could add more as desired or require, but this fits our current teams needs, importantly we've also called out the namespace in which this role is applicable. Let's apply it with `kubectl`

```bash
➜  ~ kubectl create -f team-a-deploy-role.yaml
role "team-a-deploy-role" created
```

### <u>Creating the role RoleBinding</u>

We're almost there, now we just need to bind the role to the service account we created, again the [documentation](https://kubernetes.io/docs/admin/authorization/rbac/#default-roles-and-role-bindings) is a great resource if you're struggling to understand any of the concepts. Let's take a [look](https://gist.github.com/DominicGunn/21fdc919b2fd55ef8acf0f2cc0ad86bf).

```bash
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: team-a-binding
  namespace: team-a
subjects:
- kind: ServiceAccount
  name: team-a-service-account
  namespace: team-a
roleRef:
  kind: Role
  name: team-a-deploy-role
  apiGroup: rbac.authorization.k8s.io
```

So what's happening here? A new RoleBinding is being created with the name **team-a-binding**, we're binding it to subject **team-a-service-account** and referencing the role we created **team-a-deploy-role**, all of this is happening inside of the **team-a** namespace. Let's have `kubectl` run this for us.

```bash
➜  ~ kubectl create -f team-a-role-binding.yaml
rolebinding "team-a-binding" created
```

## Share the credentials

Great, so we have a a service account that is capable of doing the deployments the team requested but now we actually need to share the credentials with **team-a**, how do we do that?

After we created the service account Kubernetes created a bunch of stuff for us, including a secret that contains the accounts CA and token. Let's have a look at the resource.

```
➜  ~ kubectl get sa team-a-service-account --namespace team-a -o json
{
    "apiVersion": "v1",
    "kind": "ServiceAccount",
    "metadata": {
        "creationTimestamp": "2018-05-20T15:59:38Z",
        "name": "team-a-service-account",
        "namespace": "team-a",
        "resourceVersion": "18409",
        "selfLink": "/api/v1/namespaces/team-a/serviceaccounts/team-a-service-account",
        "uid": "cc52ba75-5c46-11e8-9562-080027247075"
    },
    "secrets": [
        {
            "name": "team-a-service-account-token-lf4k7"
        }
    ]
}
```

We can see a secret resource has been created for us named **team-a-service-account-token-lf4k7**, we're going to use that to grab some information that we're going to share with the team.

Some of the next steps are going to include a great tool called `jq`, if you haven't played with it before I'd advise installing it and taking a look at the [documentation](https://stedolan.github.io/jq/), it's a great tool for working with JSON!

### <u>Grabbing the Certificate Authority</u>

The nature of how we've created the service account means that we authenticate against the cluster with a CA and a token, we can grab the CA and throw it into a certificate file using `kubectl`!

```
➜  ~ kubectl get secret team-a-service-account-token-lf4k7 --namespace team-a -o json | jq -r '.data["ca.crt"]' > team-a.crt

➜  ~ cat team-a.crt
ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJ....
```

### <u>Grabbing the User Token</u>

In addition to the CA, we need the bearer token that's used for authentication against the Kubernetes API, we can grab it in a very similar fashion.

```
➜  ~ kubectl get secret team-a-service-account-token-lf4k7 --namespace team-a -o json | jq -r '.data["token"]' > team-a.token

➜  ~ cat team-a.token
ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJ....
```

### Talking with Kubernetes

As the Kubernetes administrator our job is now done, we can pass the certificate and token on to the team that requires them and wipe our hands with it, but as the team, how do we now talk to the cluster?

Well first thing first, the certificate we grabbed using `kubectl` is base64 encoded, we need to decide that before we move on.

```bash
➜  ~  base64 --decode -i team-a.crt > team-a-decoded.crt
```

Great, now let's use `kubectl` to setup our local configuration so that we can talk to the cluster.

```bash
➜  ~ kubectl config set-cluster minikube --embed-certs=true --server=https://192.168.99.100:8443 --certificate-authority=team-a-decoded.crt
Cluster "minikube" set.

➜  ~ kubectl config set-credentials team-a-service-account --token=ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJ....
User "team-a-service-account" set.

➜  ~ kubectl config set-context team-a-context --cluster=minikube --user=team-a-service-account --namespace=team-a
Context "team-a-context" created.

➜  ~ kubectl  config use-context team-a-context
Switched to context "team-a-context".
```

A few details, the server `https://192.168.99.100:8443` is the address of our Minikube cluster, this may be different for you and would definitely be different in a real life scenario, your Kubernetes administrator should be able to provide this to you.

If you are the cluster administrator and you're unsure of where the master is running, you can find it easily with `cluster-info`

```bash
➜  kubectl cluster-info
Kubernetes master is running at https://192.168.99.100:8443
```

## That's all folks

There's not too much more too it. In a real world scenario you might have a couple of scripts to automate some of this setup for you, and the rules you grant to your roles might be a little tighter here and there, but this should help you hit the ground running!
