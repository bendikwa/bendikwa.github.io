---
layout: single
title:  "Deploying Grampsweb on Kubernetes"
date:   2025-02-06 14:44:00 +0100
categories: Kubernetes Grampsweb genealogy
---
I am working on documenting my family tree, and as I was searching for a self hosted alternative for the paid options (mainly MyHeritage) I came across [Grampsweb](https://www.grampsweb.org/). It describes itself as “The free, open-source genealogy system for building your family tree – together.”. This sounded perfect for my needs, and I started looking into how to get it running. The official guide uses `Docker Compose`, but let's see how we can get it running in Kubernetes.

# The official docker-compose.yaml:
If we look at the example in the [installation docs](https://www.grampsweb.org/install_setup/deployment/), we can see that the [docker-compose.yaml](https://raw.githubusercontent.com/gramps-project/gramps-web-docs/main/examples/docker-compose-base/docker-compose.yml) contains some services and a handful of volumes:

## Services:
- grampsweb
- grampsweb-celery
- grampsweb-redis
## Volumes:
- gramps_users
- gramps_index
- gramps_thumb_cache
- gramps_cache
- gramps_secret
- gramps_db
- gramps_media
- gramps_tmp

Let's start with the services. They mostly translate to Kubernetes Deployments

# Grampsweb:
{% highlight yaml %}
 grampsweb: &grampsweb
    image: ghcr.io/gramps-project/grampsweb:latest
    restart: always
    ports:
      - "80:5000"  # host:docker
    environment:
      GRAMPSWEB_TREE: "Gramps Web"  # will create a new tree if not exists
      GRAMPSWEB_CELERY_CONFIG__broker_url: "redis://grampsweb_redis:6379/0"
      GRAMPSWEB_CELERY_CONFIG__result_backend: "redis://grampsweb_redis:6379/0"
      GRAMPSWEB_RATELIMIT_STORAGE_URI: redis://grampsweb_redis:6379/1
    depends_on:
      - grampsweb_redis
    volumes:
      - gramps_users:/app/users  # persist user database
      - gramps_index:/app/indexdir  # persist search index
      - gramps_thumb_cache:/app/thumbnail_cache  # persist thumbnails
      - gramps_cache:/app/cache  # persist export and report caches
      - gramps_secret:/app/secret  # persist flask secret
      - gramps_db:/root/.gramps/grampsdb  # persist Gramps database
      - gramps_media:/app/media  # persist media files
      - gramps_tmp:/tmp
{% endhighlight %}

If we break it down it contains:
- An image: ghcr.io/gramps-project/grampsweb:latest
- A port definition: "80:5000"
- Some environment values
- Some volume mounts
- A dependency on the grampsweb_redis service.

We can translate it to a kubernetes Deployment like this:
{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grampsweb
  labels:
    app: grampsweb
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
        - name: grampsweb
          image: ghcr.io/gramps-project/grampsweb:latest
          ports:
            - containerPort: 5000
              protocol: TCP
          env:
            - name: GRAMPSWEB_CELERY_CONFIG__broker_url
              value: redis://grampsweb-redis:6379/0
            - name: GRAMPSWEB_CELERY_CONFIG__result_backend
              value: redis://grampsweb-redis:6379/0
            - name: GRAMPSWEB_RATELIMIT_STORAGE_URI
              value: redis://grampsweb-redis:6379/1
            - name: GRAMPSWEB_TREE
              value: Gramps Web
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

[grampsweb-deployment.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-deployment.yaml)

# Grampsweb Celery:

For the second service we can see that it duplicates the `grampsweb`service with some changes:

{% highlight yaml %}
 grampsweb_celery:
    <<: *grampsweb  # YAML merge key copying the entire grampsweb service config
    ports: []
    container_name: grampsweb_celery
    depends_on:
      - grampsweb_redis
    command: celery -A gramps_webapi.celery worker --loglevel=INFO --concurrency=2
{% endhighlight %}

`ports` is overridden to an empty list, `container_name`is set to `grampsweb_celery`, the dependency on grampsweb_redis is repeated, and a new startup command is added.

Translated into a kubernetes Deployment it looks like this:
{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grampsweb-celery
  labels:
    app: grampsweb-celery
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
        - name: grampsweb-celery
          image: ghcr.io/gramps-project/grampsweb:latest
          args:
          - celery
          - -A
          - gramps_webapi.celery worker
          - --loglevel=INFO
          - --concurrency=2
          env:
            - name: GRAMPSWEB_CELERY_CONFIG__broker_url
              value: redis://grampsweb-redis:6379/0
            - name: GRAMPSWEB_CELERY_CONFIG__result_backend
              value: redis://grampsweb-redis:6379/0
            - name: GRAMPSWEB_RATELIMIT_STORAGE_URI
              value: redis://grampsweb-redis:6379/1
            - name: GRAMPSWEB_TREE
              value: Gramps Web
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

[grampsweb-celery-deployment.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-celery-deployment.yaml)

# Grampsweb Redis

The `grampsweb_redis` service is just a Deployment of the official redis image:
{% highlight yaml %}
 grampsweb_redis:
    image: docker.io/library/redis:7.2.4-alpine
    container_name: grampsweb_redis
    restart: always
{% endhighlight %}

Translated to a Kubernetes Deployment it would look like this: (OBS we change the underscore in the name to a dash to follow kubernetes naming conventions)
{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grampsweb-redis
  labels:
    app: grampsweb-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grampsweb-redis
  template:
    metadata:
      labels:
        app: grampsweb-redis
    spec:
      containers:
        - name: grampsweb-redis
          image: docker.io/library/redis:7.2.4-alpine
{% endhighlight %}

[grampsweb-redis-deployment.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-redis-deployment.yaml)

# Grampsweb Service
In addition to the Deployment, we need a `Kubernetes` Service for Grampsweb to route traffic. (not to be confused with `Docker compose`services)
In `docker-compose.yaml` we see that the container serves content on port 5000, but port 80 should be used on the host. We set up the same thing in the Service:
{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: grampsweb
  labels:
    app: grampsweb
spec:
  ports:
    - name: "80"
      port: 80
      targetPort: 5000
  selector:
    app: grampsweb
{% endhighlight %}

This Service will default to the type ClusterIP and will only be accessible inside the cluster.

[grampsweb-service.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-service.yaml)

# Grampsweb Redis Service

The Service for `Redis` is similar, but we do not translate the port:
{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: grampsweb-redis
  labels:
    app: grampsweb-redis
spec:
  ports:
    - name: "redis"
      port: 6379
  selector:
    app: grampsweb-redis
{% endhighlight %}

[grampsweb-redis-service.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-redis-service.yaml)

# Volumes
We also need to create the volumes defined in docker-compose.yaml.
In Kubernetes persistent volumes have two parts. A PersistentVolumeClaim (PVC) and a PersistentVolume (PV)
Depending on what storage provider you are using, you might need to configure both, but if you have a storage-class in your cluster with auto provisioning, you just need the PVCs.
I am using the [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) in my cluster and it will take care of the PVs for me.
The PVCs all look very similar, and will be on the form:

{% highlight yaml %}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <volume_name>
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
{% endhighlight %}

Repeat for all the volumes:
- gramps-users
- gramps-index
- gramps-thumb-cache
- gramps-cache
- gramps-secret
- gramps-db
- gramps-media
- gramps-tmp

You can add multiple Kubernetes objects to a file, and to keep it practical, you can add all the volumes to one file. See [grampsweb-all-pvcs.yaml]({{ site.url }}{{ site.baseurl }}/assets/files/grampsweb/grampsweb-all-pvcs.yaml)

# Putting it all together

If we store the different Deployment and Service definitions in separate files, and put all the volumes in a one, we will end up with something like this:
{% highlight shell %}
ls
grampsweb-celery-deployment.yaml  grampsweb-redis-deployment.yaml  grampsweb-service.yaml
grampsweb-deployment.yaml         grampsweb-redis-service.yaml     grampsweb-all-pvcs.yaml
{% endhighlight %}

Deploying the definitions with kubectl as they are will deploy them in the default namespace, you can change that by adding the namespace flag to the command.
{: .notice--warning}


Since there is no dependency declaration in native kubernetes objects, it is practical to apply them in separate steps:
1. The Volumes
2. The Redis Deployment and Service
3. The Grampsweb Celery Deployment
4. The Grampsweb Deployment and Service

## Deploy the Volumes
With all the PVC definitions in one file, you can apply them all at once:
{% highlight shell %}
kubectl apply -f grampsweb-all-pvcs.yaml
persistentvolumeclaim/gramps-cache created
persistentvolumeclaim/gramps-db created
persistentvolumeclaim/gramps-index created
persistentvolumeclaim/gramps-media created
persistentvolumeclaim/gramps-secret created
persistentvolumeclaim/gramps-thumb-cache created
persistentvolumeclaim/gramps-tmp created
persistentvolumeclaim/gramps-users created
{% endhighlight %}

## Deploy Redis
{% highlight shell %}
kubectl apply -f grampsweb-redis-deployment.yaml -f grampsweb-redis-service.yaml
deployment.apps/grampsweb-redis created
service/grampsweb-redis created
{% endhighlight %}

Tail the logs and wait for Redis to finish start up
{% highlight shell %}
... Ready to accept connections tcp
{% endhighlight %}

## Deploy Gampsweb Celery
{% highlight shell %}
kubectl apply -f grampsweb-celery-deployment.yaml
deployment.apps/grampsweb-celery created
{% endhighlight %}

Tail the logs and wait for grampsweb-celery to finish start up
{% highlight shell %}
... celery@grampsweb-celery-<pod_name> ready.
{% endhighlight %}

## Deploy Grampsweb
{% highlight shell %}
kubectl apply -f grampsweb-deployment.yaml -f grampsweb-service.yaml
deployment.apps/grampsweb created
service/grampsweb created
{% endhighlight %}

# Testing
Your Grampsweb instance should now be up and running.
To test it at this stage you can forward port 8080 on localhost to the internal kubernetes Service:
{% highlight shell %}
kubectl port-forward service/grampsweb 8080:80
{% endhighlight %}

Remember to include the namespace in the port-forward command, if you deployed to something else than `default`
{: .notice--warning}

Browsing to <http://localhost:8080> should now give you the “Welcome to Gramps Web” page, prompting you to set up an admin user.

Among some warnings I got two errors in the logs: `assertion 'GDK_IS_SCREEN (screen)' failed` and `Error parsing list of recent DBs from file /root/.gramps/recent-files-gramps.xml`. Both can be safely ignored according to this:  <https://gramps.discourse.group/t/errors-seen-in-logs/4774>
{: .notice--info}

# Ingress traffic

In my setup, I use Traefik as `Ingress Controller` and Certbot to add TLS, so all I need is to an an ingress and set up DNS to finalize the setup:
{% highlight yaml %}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grampsweb
  annotations:
    cert-manager.io/cluster-issuer: <ISSUER>
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  tls:
    - hosts:
        - <HOSTNAME>
      secretName: <SECRET_NAME>
  rules:
    - host: <HOSTNAME>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name:  grampsweb
                port:
                  number: 80
{% endhighlight %}

Replace ISSUER, HOSTNAME, and SECRET_NAME with correct values for your setup.

# Profit!

There are, of course, many things you can do to improve the setup. Securing the redis instance is one, moving common `Grampsweb` config to a `Config Map` is another, but this covers the basic exercise of getting your instance running.