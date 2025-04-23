---
layout: single
title:  "Improving Grampsweb kubernetes setup with Config Map and secret"
date:   2025-04-23 11:32:00 +0100
categories: Kubernetes Grampsweb genealogy
---
Let's improve the basic setup we did in [Deploying Grampsweb on Kubernetes](https://complicated.bendikwa.com/kubernetes/grampsweb/genealogy/deploying-grampsweb-on-kubernetes/) by moving the configuration into a configmap that can be shared between Grampsweb and Celery

# The ConfigMap:
A Kubernetes ConfigMap object holds `non-confidential` key-value pairs that can be consumed from Pods. 
One way of consuming it is to [mount them as environment variables](https://kubernetes.io/docs/concepts/configuration/configmap/#using-configmaps-as-environment-variables).

Continuing the setup for Grampsweb on Kubernetes, we can improve it by putting all the environment variables defined in [docker-compose](https://raw.githubusercontent.com/gramps-project/gramps-web-docs/main/examples/docker-compose-base/docker-compose.yml) into a new ConfigMap definition:

{% highlight yaml %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: grampsweb
data:
  GRAMPSWEB_CELERY_CONFIG__broker_url: "redis://grampsweb-redis:6379/0"
  GRAMPSWEB_CELERY_CONFIG__result_backend: "redis://grampsweb-redis:6379/0"
  GRAMPSWEB_RATELIMIT_STORAGE_URI: "redis://grampsweb-redis:6379/1"
  GRAMPSWEB_TREE: "Gramps Web"
{% endhighlight %}

[grampsweb-configmap.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-configmap.yaml)

# Mounting the ConfigMap:

We can now replace the individual environment variable definitions in [grampsweb-deployment.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-deployment.yaml) and [grampsweb-celery-deployment.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-celery-deployment.yaml) with an entry to mount the ConfigMap:

## Grampsweb

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
          image: ghcr.io/gramps-project/grampsweb:latest
          name: grampsweb
          ports:
            - containerPort: 5000
              protocol: TCP
          volumeMounts:
            - mountPath: /app/users
              name: gramps-users
            - mountPath: /app/indexdir
              name: gramps-index
            - mountPath: /app/thumbnail_cache
              name: gramps-thumb-cache
            - mountPath: /app/cache
              name: gramps-cache
            - mountPath: /app/secret
              name: gramps-secret
            - mountPath: /root/.gramps/grampsdb
              name: gramps-db
            - mountPath: /app/media
              name: gramps-media
            - mountPath: /tmp
              name: gramps-tmp
      volumes:
        - name: gramps-users
          persistentVolumeClaim:
            claimName: gramps-users
        - name: gramps-index
          persistentVolumeClaim:
            claimName: gramps-index
        - name: gramps-thumb-cache
          persistentVolumeClaim:
            claimName: gramps-thumb-cache
        - name: gramps-cache
          persistentVolumeClaim:
            claimName: gramps-cache
        - name: gramps-secret
          persistentVolumeClaim:
            claimName: gramps-secret
        - name: gramps-db
          persistentVolumeClaim:
            claimName: gramps-db
        - name: gramps-media
          persistentVolumeClaim:
            claimName: gramps-media
        - name: gramps-tmp
          persistentVolumeClaim:
            claimName: gramps-tmp
{% endhighlight %}

[grampsweb-deployment-with-configmap.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-deployment-with-configmap.yaml)

## Grampsweb Celery:

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
          image: ghcr.io/gramps-project/grampsweb:latest
          name: grampsweb-celery
          volumeMounts:
            - mountPath: /app/users
              name: gramps-users
            - mountPath: /app/indexdir
              name: gramps-index
            - mountPath: /app/thumbnail_cache
              name: gramps-thumb-cache
            - mountPath: /app/cache
              name: gramps-cache
            - mountPath: /app/secret
              name: gramps-secret
            - mountPath: /root/.gramps/grampsdb
              name: gramps-db
            - mountPath: /app/media
              name: gramps-media
            - mountPath: /tmp
              name: gramps-tmp
      volumes:
        - name: gramps-users
          persistentVolumeClaim:
            claimName: gramps-users
        - name: gramps-index
          persistentVolumeClaim:
            claimName: gramps-index
        - name: gramps-thumb-cache
          persistentVolumeClaim:
            claimName: gramps-thumb-cache
        - name: gramps-cache
          persistentVolumeClaim:
            claimName: gramps-cache
        - name: gramps-secret
          persistentVolumeClaim:
            claimName: gramps-secret
        - name: gramps-db
          persistentVolumeClaim:
            claimName: gramps-db
        - name: gramps-media
          persistentVolumeClaim:
            claimName: gramps-media
        - name: gramps-tmp
          persistentVolumeClaim:
            claimName: gramps-tmp

{% endhighlight %}

[grampsweb-celery-deployment-with-configmap.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-celery-deployment-with-configmap.yaml)

# Deploying:

## ConfigMap:
First we must deploy the ConfigMap. Remember to deploy it to the same namespace as Grampsweb, using the `-n` flag with `kubectl` if you want to a different namespace than `default`:

{% highlight shell %}
kubectl apply -f grampsweb-configmap.yaml
configmap/grampsweb created
{% endhighlight %}

## Deploy Gampsweb Celery
{% highlight shell %}
kubectl apply -f grampsweb-celery-deployment-with-configmap.yaml
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
kubectl apply -f grampsweb-deployment-with-configmap.yaml
deployment.apps/grampsweb created
{% endhighlight %}

# Profit!

From now on we have a single place to put configuration for our Grampsweb instance. Remember to restart both the Celery and the Grampsweb pods when updating the ConfigMap.
The ConfigMap is made for `non-confidential` data. For sensitive values, use a [secret] https://kubernetes.io/docs/concepts/configuration/secret/
