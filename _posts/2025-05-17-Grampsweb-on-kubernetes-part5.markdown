---
layout: single
title:  "Grampsweb on kubernetes - Part 5: Main family tree database"
date:   2025-05-17 08:30:00 +0100
categories: Kubernetes Grampsweb genealogy postgres
---
In [the previous part]({% link _posts/2025-05-17-Grampsweb-on-kubernetes-part4.markdown %}) we configured `Grampsweb` to use external `PostgreSQL` databases for the `Users` and `Search index`. In this part we will replace the default `SQLight` database for the `family tree`.

This guide was written for Gramps version 5. (up to Grampsweb version v25.4.1). At the time of writing the PosgreSQL addon does not work with Gramps 6 (Grampsweb version v25.5.0 and v25.5.1). A fix is committed, and it should be working again once v25.5.2 is released.
{: .notice--warning}

This guide is written assuming a new, clean setup. If you already have data in your family tree that you want to keep, you **must** make sure that you have a **working backup**, and **know how to restore it**.
{: .notice--danger}

Before following this guide you must make sure that you have a host or service where you can create `PostgreSQL` databases, and that it is accessible from the pods running `Grampsweb`. I have gotten various errors connected to `locale` when testing this, but I have had most success with matching the locale of the database with the `locale` of the system that runs the `Gramps` instance producing the .gramps file used in the initial `database` initialization as described <A href="#prepare-to-initialize-the-database">below</A>. Out of the box the `Grampsweb` docker image uses `en_US.utf8`.
{: .notice--info}

# The main database
From the [Grampsweb docs](https://www.grampsweb.org/install_setup/postgres/#configuring-web-api-for-use-with-the-database) we can see that we need to add some more environment values to configure `Grampsweb` to use a `PostgreSQL` backend for the main database:

{% highlight yaml %}
  GRAMPSWEB_POSTGRES_USER: [db_user_name]
  GRAMPSWEB_POSTGRES_PASSWORD: [db_user_password]
{% endhighlight %}

We can use the same user as for the `search`and `user` databases, just give it sufficient privileges to the `database` we create later. E.g. set the `user` as `owner` of the `database`.
{: .notice}

We also need to add the `hostname` and `port` of our `PostgreSQL` instance:

{% highlight yaml %}
  GRAMPSWEB_DATABASE_HOST: [host]
  GRAMPSWEB_DATABASE_PORT: [port]
{% endhighlight %}

To be on the safe side, we also add the `database_backend` and `new_db_backend`, but I am not 100% certain that they are needed:

{% highlight yaml %}
  GRAMPSWEB_NEW_DB_BACKEND: "postgresql"
  GRAMPSWEB_DATABASE_BACKEND: "postgresql"
{% endhighlight %}

The value of `POSTGRES_PASSWORD` is sensitive, so we add it to our `secret`.

{% highlight yaml %}
apiVersion: v1
kind: Secret
metadata:
  name: grampsweb
type: Opaque
stringData:
  GRAMPSWEB_SECRET_KEY: GzZNcINdZSxrP9QUexOMYjWKJ_UnVt3tozQME0uY8KM
  GRAMPSWEB_USER_DB_URI: postgresql://[user]:[password]@[host]:[port]/grampswebuser
  SEARCH_INDEX_DB_URI: postgresql://[user]:[password]@[host]:[port]/grampswebsearch
  GRAMPSWEB_POSTGRES_PASSWORD: [db_user_password]
{% endhighlight %}

[grampsweb-secret-with-database-2.yaml]({% link /assets/files/grampsweb/grampsweb-secret-with-database-2.yaml %})

The rest of the values are `non-sensitive` and we can add them to our `ConfigMap`.

After these additions, the complete `ConfigMap` looks like this:

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
  GRAMPSWEB_POSTGRES_USER: [db_user_name]
  GRAMPSWEB_DATABASE_HOST: [host]
  GRAMPSWEB_DATABASE_PORT: [port]
  GRAMPSWEB_NEW_DB_BACKEND: "postgresql"
  GRAMPSWEB_DATABASE_BACKEND: "postgresql"
{% endhighlight %}

[grampsweb-configmap-with-database.yaml]({% link /assets/files/grampsweb/grampsweb-configmap-with-database.yaml %})

# Deploying
## Prepare to initialize the database
`Grampsweb` will not use the external `PostgreSQL` as backend before it is initialized.
To  do this we will adapt the method used in the [Grampsweb docs](https://www.grampsweb.org/install_setup/postgres/#importing-a-gramps-family-tree) to import a `Gramps XML file`.

I have tried many ways of getting `Grampsweb` to initialize a new `family tree` using the `PostgreSQL` backend, and this is the only way I have got it working. I am open to there being a better/simpler way, but for now, this is how I have done it.
{: .notice--warning}

1. Prepare the `Gramps XML file`. If you don't already have a `family tree` you want to import, you can just export one from the `Grampsweb` web-interface. The format can probably be of any `Grampsweb`-supported type, but I have only tested with `Gramps XML`(`.gramps`) files.
2. Copy the `Gramps XML file` to one of the shared volumes mounted in the `Grampsweb` and `Celery` pods. E.g. the `gramps-tmp` volume.
3. Make sure that you have the `GRAMPSWEB_TREE` variable set to a different value than what you want your final tree to be named. If you have followed my setup, it will probably be set to "Gramps Web". *If you want to use the name currently in use, you must:* 
    1. Change the value of `GRAMPSWEB_TREE` to something else in the `ConfigMap`
    2. Deploy the `ConfigMap`
    3. Restart the `Grampsweb` and `Celery` `Deployments`
    4. Remove the folder <A href="#identify-backend-folders-in-the-gramps-db-volume">corresponding</A> to the `tree` with the desired name from the `gramps-db` volume. **WARNING: This will delete any data in the `family tree` stored there.**
4. Decide on a new name for your `family tree`, we will use this in the next step.
5. Create a `database` in the `Postgres cluster` for the `family tree` to use. The name must be equal to the new `family tree`. For this example let's call the `database` "tree".
6. Give the `user` sufficient privileges to the "tree" `database`. E.g. set the `user` as `owner` of the `database`.
7. Create a `Kubernetes` `Job` to run the import command. Substitute the `new_tree_name` with the name for the `family tree` (in our example "tree"), and `absolute_path_to_the_.gramps_file_to_import` with the path to your `Gramps XML file` as seen inside the pod. E.g. if you copied it to the `gramps-tmp`, you can access it at `/tmp/<filename>`
We mount the `Secret` and `ConfigMap` the same way as in the `Deployments` so that all needed values are present.

{% highlight yaml %}
apiVersion: batch/v1
kind: Job
metadata:
  name: grampsweb-import
spec:
  template:
    metadata:
      labels:
        app: grampsweb-import
    spec:
      containers:
      - name: grampsweb-import
        image: ghcr.io/gramps-project/grampsweb:latest
        command: []
        args:
        - gramps
        - -C
        - <new_tree_name>
        - -i
        - <absolute_path_to_the_.gramps_file_to_import>
        - --config=database.backend:$(GRAMPSWEB_DATABASE_BACKEND)
        - --config=database.host:$(GRAMPSWEB_DATABASE_HOST)
        - --config=database.port:$(GRAMPSWEB_DATABASE_PORT)
        - --username=$(GRAMPSWEB_POSTGRES_USER)
        - --password=$(GRAMPSWEB_POSTGRES_PASSWORD)
        envFrom:
        - configMapRef:
            name: grampsweb
        - secretRef:
            name: grampsweb
        volumeMounts:
        - mountPath: /app/thumbnail_cache
          name: gramps-thumb-cache
        - mountPath: /app/cache
          name: gramps-cache
        - mountPath: /root/.gramps/grampsdb
          name: gramps-db
        - mountPath: /app/media
          name: gramps-media
        - mountPath: /tmp
          name: gramps-tmp
      restartPolicy: Never
      volumes:
      - name: gramps-thumb-cache
        persistentVolumeClaim:
          claimName: gramps-thumb-cache
      - name: gramps-cache
        persistentVolumeClaim:
          claimName: gramps-cache
      - name: gramps-db
        persistentVolumeClaim:
          claimName: gramps-db
      - name: gramps-media
        persistentVolumeClaim:
          claimName: gramps-media
      - name: gramps-tmp
        persistentVolumeClaim:
          claimName: gramps-tmp
  backoffLimit: 0
{% endhighlight %}

[grampsweb-import-job.yaml]({% link /assets/files/grampsweb/grampsweb-import-job.yaml %})

## Deploy Secret and ConfigMap
As always, remember to set the `namespace` by adding the `-n <namespace>` flag to `kubectl` if you want to deploy to a different `namespace` than `default`
{: .notice--info}

{% highlight shell %}
kubectl apply -f grampsweb-secret-with-database.yaml -f grampsweb-configmap-with-database.yaml
configmap/grampsweb configured
secret/grampsweb configured
{% endhighlight %}

## Deploy import-job
Deploy the job and tail the logs to see that it finished successfully. The namespace must be the same as all the other `Grampsweb`components.
{% highlight shell %}
kubectl apply -f grampsweb-import-job.yaml
job.batch/grampsweb-import created
{% endhighlight %}

Tail the logs to see that the `job` finishes successfully:
{% highlight shell %}
kubectl -n gramps logs -f job.batch/grampsweb-import
...
Opened successfully!
Importing: file /tmp/empty-tree.gramps, format gramps.
100%Cleaning up.
{% endhighlight %}

If you get error messages instead of the message above, you can check the <A href="#debugging">debugging</A> section below.
{: .notice-warning}

After the `job` has finished, we can delete it:
{% highlight shell %}
kubectl delete -n gramps job.batch/grampsweb-import
job.batch "grampsweb-import" deleted
{% endhighlight %}

## Grampsweb and Celery
After the `import-job` has succeeded, we need to change the active tree in use.
We do this by changing the `GRAMPSWEB_TREE` value in our `ConfigMap` to match the value we picked for the <new_tree_name>. In this example, we used "tree".

{% highlight yaml %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: grampsweb
data:
  GRAMPSWEB_CELERY_CONFIG__broker_url: "redis://grampsweb-redis:6379/0"
  GRAMPSWEB_CELERY_CONFIG__result_backend: "redis://grampsweb-redis:6379/0"
  GRAMPSWEB_RATELIMIT_STORAGE_URI: "redis://grampsweb-redis:6379/1"
  GRAMPSWEB_TREE: "tree"
  GRAMPSWEB_POSTGRES_USER: [db_user_name]
  GRAMPSWEB_DATABASE_HOST: [host]
  GRAMPSWEB_DATABASE_PORT: [port]
  GRAMPSWEB_NEW_DB_BACKEND: "postgresql"
  GRAMPSWEB_DATABASE_BACKEND: "postgresql"
{% endhighlight %}

[grampsweb-configmap-with-database-2.yaml]({% link /assets/files/grampsweb/grampsweb-configmap-with-database-2.yaml %})

{% highlight shell %}
kubectl apply grampsweb-configmap-with-database-2.yaml
configmap/grampsweb configured
{% endhighlight %}

After deploying the `ConfigMap` we must restart `Grampsweb` and `Celery`:
{% highlight shell %}
kubectl rollout restart deployment/grampsweb deployment/grampsweb-celery
deployment.apps/grampsweb restarted
deployment.apps/grampsweb-celery restarted
{% endhighlight %}

Tail the logs of `Celery` and wait for it to become ready:
{% highlight shell %}
kubectl -logs -f deployment/grampsweb-celery
...
<timestamp> celery@grampsweb-celery-fbf679f5c-m96md ready.
{% endhighlight %}

# Verify
We can verify by using the `Grampsweb` web-interface and going to "Profile"/"Settings" -> "System Information" -> "Tree Information". We should see that the `ID` is a short sequence on the form `0000x000`, not a longer hash like sequence e.g. `b2a81742-fc73-4b31-9f4a-f561850945ae`. If you don't see a change in the `ID` after restarting, try to close the browser tab and open a new one. The content is sometimes cached.

We can also verify by logging in to our `tree` database and see that it is populated with tables like: `person` and `family`

# Profit!
We now have our main `family tree` database separated out into an external instance that can easily be backed up and restored if needed.

We can NOT remove the `gramps-db` volume from the `Grampsweb` and `Celery` `Deployments` since it is used to store config telling `Grampsweb` about the database backend.
{: .notice--warning}

# Debugging

## Identify backend folders in the `gramps-db` volume
You can find out which `folders` are associated with which `tree names` in the `gramps-db` volume by looking at the `name.txt` file inside it. It contains the name of that `tree`.
{: .notice--success}

## Password authentication failed for user `user`
{% highlight shell %}
ERROR: Database connection failed.
connection to server at "<host>" (<IP>), port <port> failed: FATAL:  password authentication failed for user "<user>"
{% endhighlight %}

The credentials for the `PostgreSQL` user is not correct.

To fix this:
1. Correct the `user` and `password` for your `PostgreSQL` user.
  - Remember to redeploy the `Secret` and/or `ConfigMap`
2. Remove the folder <A href="#identify-backend-folders-in-the-gramps-db-volume">corresponding</A> to the `tree` with the desired name from the `gramps-db` volume. **WARNING: This will delete any data in the `family tree` stored there.**
3. First delete, then redeploy the `import-job`.

## Error: Family Tree `new_tree_name` already exists.
{% highlight shell %}
Error: Family Tree <new_tree_name> already exists.
The '-C' option cannot be used.
{% endhighlight %}

This error can be seen on a couple of scenarios:
1. Using the same name for <new_tree_name> as `GRAMPSWEB_TREE` (in the deployed `ConfigMap`)
2. A folder already exists in the `gramps-db` volume containing a `database` configuration for a `family tree` with that name.

To fix this:
1. Re-deploy your `ConfigMap` with correct values.
2. Restart the `Grampsweb` and `Celery` `Deployments`
3. Remove the folder <A href="#identify-backend-folders-in-the-gramps-db-volume">corresponding</A> to the `tree` with the desired name from the `gramps-db` volume. **WARNING: This will delete any data in the `family tree` stored there.**
4. First delete, then redeploy the `import-job`

## FATAL:  database `new_tree_name` does not exist
{% highlight shell %}
ERROR: Database connection failed.
connection to server at "<host>" (<IP>), port <port> failed: FATAL:  database "<new_tree_name>" does not exist
{% endhighlight %}

A database with the same name as <new_tree_name> is not present in the `PostgreSQL instance`

To fix this:
1. Create the database in the `PostgreSQL` cluster.
2. Set the `user` as the `owner` of the `database`.
3. Remove the folder <A href="#identify-backend-folders-in-the-gramps-db-volume">corresponding</A> to the `tree` with the desired name from the `gramps-db` volume. **WARNING: This will delete any data in the `family tree` stored there.**
4. First delete, then redeploy the `import-job`.

## Locked by noUSER@grampsweb-import-XXXX
{% highlight shell %}
Database is locked, cannot open it!
Info: Locked by noUSER@grampsweb-import-XXXX
{% endhighlight %}

The import has failed partially, and not been cleaned up
It is possible to instruct `gramps` to ignore the lock, but it would fail with a "Family Tree 'tree' already exists" error anyway.

To fix this:
1. Remove the folder <A href="#identify-backend-folders-in-the-gramps-db-volume">corresponding</A> to the `tree` with the desired name from the `gramps-db` volume. **WARNING: This will delete any data in the `family tree` stored there.**
2. First delete, then redeploy the `import-job`.
