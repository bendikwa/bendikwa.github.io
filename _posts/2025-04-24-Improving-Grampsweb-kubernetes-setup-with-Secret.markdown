---
layout: single
title:  "Improving Grampsweb kubernetes setup with Secret"
date:   2025-04-24 08:15:00 +0100
categories: Kubernetes Grampsweb genealogy
---
We have [previously]({% link _posts/2025-04-23-Improving-Grampsweb-kubernetes-setup-with-Config-Map.markdown %}) moved all `non-confidential` configuration to a `ConfigMap`, now we will add a [Secret](https://kubernetes.io/docs/concepts/configuration/secret/).

# The flask secret key
Looking at the [docker-entrypoint.sh](https://github.com/gramps-project/gramps-web-api/blob/8f1ef9359cec56b4dbeb229bf65bbf90a7386393/docker-entrypoint.sh#L5) it attempts to read the `flask secret key` from `GRAMPSWEB_SECRET_KEY` before using a file on the shared volume mounted at `/app/secret`
If we provide the secret in `GRAMPSWEB_SECRET_KEY`, we can simplify the setup, and get rid of the `gramps-secret`volume.

# The Secret:
A [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) is an object that contains a small amount of sensitive data such as a password, a token, or a key. The default Secret type (`Opaque`) can hold a set of key-value pairs, and like the ConfigMap, it can be consumed by `Pods`by [mounting them as environment variables](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables).

To generate the actual data for the secret, we can use the same command as is done in the [docker-entrypoint.sh](https://github.com/gramps-project/gramps-web-api/blob/8f1ef9359cec56b4dbeb229bf65bbf90a7386393/docker-entrypoint.sh#L11):

{% highlight shell %}
python3 -c "import secrets;print(secrets.token_urlsafe(32))"
GzZNcINdZSxrP9QUexOMYjWKJ_UnVt3tozQME0uY8KM
{% endhighlight %}

Then we can create a `Secret` holding the `GRAMPSWEB_SECRET_KEY` like this:

{% highlight yaml %}
apiVersion: v1
kind: Secret
metadata:
  name: grampsweb
  namespace: gramps
type: Opaque
stringData:
  GRAMPSWEB_SECRET_KEY: GzZNcINdZSxrP9QUexOMYjWKJ_UnVt3tozQME0uY8KM
{% endhighlight %}

[grampsweb-secret.yaml]({% link /assets/files/grampsweb/grampsweb-secret.yaml %})

# Mounting the Secret:

We add the `Secret` to the deployments the same way as we did for the `ConfigMap`, and we can now also remove the `gramps-secret` mount:

## Grampsweb:

### Add secretRef:
{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: grampsweb
  name: grampsweb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grampsweb
  template:
    metadata:
      labels:
        app: grampsweb
    spec:
      containers:
        - envFrom:
          - configMapRef:
              name: grampsweb
          - secretRef:
              name: grampsweb
          image: ghcr.io/gramps-project/grampsweb:latest
          name: grampsweb
          ports:
            - containerPort: 5000
              protocol: TCP
...
{% endhighlight %}


### Remove gramps-secret mount:
{% highlight yaml %}
...
            - mountPath: /app/secret
              name: gramps-secret
...
{% endhighlight %}
[grampsweb-deployment-with-secret.yaml]({% link /assets/files/grampsweb/grampsweb-deployment-with-secret.yaml %})

## Grampsweb Celery:

### Add secretRef:
{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: grampsweb-celery
  name: grampsweb-celery
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grampsweb-celery
  template:
    metadata:
      labels:
        app: grampsweb-celery
    spec:
      containers:
        - args:
            - celery
            - -A
            - gramps_webapi.celery
            - worker
            - --loglevel=INFO
            - --concurrency=2
          envFrom:
          - configMapRef:
              name: grampsweb
          - secretRef:
              name: grampsweb
...

{% endhighlight %}


### Remove gramps-secret mount:
{% highlight yaml %}
...
            - mountPath: /app/secret
              name: gramps-secret
...
{% endhighlight %}


[grampsweb-celery-deployment-with-secret.yaml]({% link /assets/files/grampsweb/grampsweb-celery-deployment-with-secret.yaml %})

# Deploying:

## Secret:
First we must deploy the Secret. Remember to deploy it to the same namespace as Grampsweb, using the `-n` flag with `kubectl` if you want to a different namespace than `default`:

{% highlight shell %}
kubectl apply -f grampsweb-secret.yaml
secret/grampsweb created
{% endhighlight %}

## Deploy Gampsweb Celery
{% highlight shell %}
kubectl apply -f grampsweb-celery-deployment-with-secret.yaml
deployment.apps/grampsweb-celery created
{% endhighlight %}

Tail the logs of the container and wait for grampsweb-celery to finish start up
{% highlight shell %}
kubectl get pod -l app=grampsweb-celery
NAME                               READY   STATUS    RESTARTS   AGE
grampsweb-celery-fbf679f5c-m96md   1/1     Running   0          10m
kubectl logs -f grampsweb-celery-fbf679f5c-m96md
<timestamp> celery@grampsweb-celery-fbf679f5c-m96md ready.
{% endhighlight %}

## Deploy Grampsweb
{% highlight shell %}
kubectl apply -f grampsweb-deployment-with-secret.yaml
deployment.apps/grampsweb created
{% endhighlight %}

# Profit!

From now on we also have a single place to put sensitive configuration for our Grampsweb instance. Remember to restart both the Celery and the Grampsweb pods when updating the ConfigMap.
