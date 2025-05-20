---
layout: single
title:  "Grampsweb on kubernetes - Part 4: User and Search index databases"
date:   2025-05-17 08:15:00 +0100
categories: Kubernetes Grampsweb genealogy postgres
---
In the previous guides we have added a [ConfigMap]({% link _posts/2025-04-23-Improving-Grampsweb-kubernetes-setup-with-a-Config-Map.markdown %}) and a [Secret]({% link _posts/2025-04-24-Improving-Grampsweb-kubernetes-setup-with-a-Secret.markdown %}) to our Grampsweb instance. Now we will replace the file-based `SQLite` databases used by `Grampsweb` with external `PostgreSQL` databases. This will be done using the [Gramps PostgreSQL Addon](https://gramps-project.org/wiki/index.php/Addon:PostgreSQL) which is already installed in the [Grampsweb container](https://github.com/gramps-project/gramps-web-api/blob/8f1ef9359cec56b4dbeb229bf65bbf90a7386393/Dockerfile#L56).

This guide is written assuming a new, clean setup. If you already have data in your family tree that you want to keep, you **must** make sure that you have a **working backup**, and **know how to restore it**.
{: .notice--danger}

Before following this guide you must make sure that you have a host or service where you can create `PostgreSQL` databases, and that it is accessible from the pods running Grampsweb. In my experience, the locale of the databases need to match the locale of the running Grampsweb instance. Out of the box this is `en_US.utf8`.
{: .notice--info}

There are tree separate `SQLite` databases used by `Grampsweb` that we can replace with `PostgreSQL` databases:
1. The **User** database
2. The **Search index**
3. The **main database** used for storing the `family tree`

**1** and **2** will be covered in this part, **3** will be covered in [part 2]({% link _posts/2025-05-17-Grampsweb-on-kubernetes-part5.markdown %})

# The User database
First create a `user` and `database` in the `Postgres cluster` for `Grampsweb` to use. They can be named anything you want. For this example let's call the `user` "grampsweb" and the `database` "users".

Remember to give the user sufficient privileges to the "users" `database`. E.g. set the `user` as `owner`of the `database`.
{: .notice}

From the [Grampsweb documentation](https://www.grampsweb.org/install_setup/postgres/#using-a-postgresql-database-for-the-user-database) we can see that all we need to do is to add an environment variable named `USER_DB_URI` with the Postgres [connection URI](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING-URIS) as value.

The format of the `connection URI` is `postgresql://[user]:[password]@[host]:[port][/dbname]`, and since it includes the password for the database, this should be added to our `Secret`.

{% highlight yaml %}
apiVersion: v1
kind: Secret
metadata:
  name: grampsweb
type: Opaque
stringData:
  GRAMPSWEB_SECRET_KEY: GzZNcINdZSxrP9QUexOMYjWKJ_UnVt3tozQME0uY8KM
  GRAMPSWEB_USER_DB_URI: postgresql://grampsweb:[password]@[host]:[port]/users
{% endhighlight %}

Replace `password` with the password of the `user` you created. Replace `host` and `port` with the hostname/IP address to your `Postgres instance` and the port it is exposing. The standard port for `PostgreSQL` is `5432`.

# The Search index database
The database for the `Search index` is set up in the same way as the `User database`
First create a `database` in the `Postgres cluster` for the `search index` to use. For this example let's call the `database` "search".

We can use the same user as above, just give it sufficient privileges to the "search" `database`. E.g. set the `user` as `owner`of the `database`.
{: .notice}

Once again we can see in the [Grampsweb documentation](https://www.grampsweb.org/install_setup/postgres/#using-a-postgresql-database-for-the-search-index) that all we need to do is to add an environment variable named `SEARCH_INDEX_DB_URI` with the Postgres [connection URI](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING-URIS) as value.

Since this also includes the password for the database, it should be added to our `Secret`.

{% highlight yaml %}
apiVersion: v1
kind: Secret
metadata:
  name: grampsweb
type: Opaque
stringData:
  GRAMPSWEB_SECRET_KEY: GzZNcINdZSxrP9QUexOMYjWKJ_UnVt3tozQME0uY8KM
  GRAMPSWEB_USER_DB_URI: postgresql://grampsweb:[password]@[host]:[port]/users
  SEARCH_INDEX_DB_URI: postgresql://grampsweb:[password]@[host]:[port]/search
{% endhighlight %}

As with the `User database`, we must replace `password` with the password of our `user`. Replace `host` and `port` with the hostname/IP address to your `Postgres instance` and the port it is exposing. The standard port for `PostgreSQL` is `5432`.

# Deploying User and Search database changes

Remember to set the `namespace` by adding the `-n <namespace>` flag to `kubectl` for all the following commands if you want to deploy to a different `namespace` than `default`
{: .notice--info}

## Deploy Secret
First we deploy the changes to our `Secret`: [grampsweb-secret-with-database-1.yaml]({% link /assets/files/grampsweb/grampsweb-secret-with-database-1.yaml %})

{% highlight shell %}
kubectl apply -f grampsweb-secret-with-database-1.yaml
secret/grampsweb configured
{% endhighlight %}

## Restart Gampsweb and Celery
The `Grampsweb` and `Celery` instances must be restarted to have them pick up the new configuration values in the `Secret`.

We can trigger a restart of `Deployments` with the `kubectl rollout restart` command:
{% highlight shell %}
kubectl rollout restart deployment/grampsweb deployment/grampsweb-celery
deployment.apps/grampsweb restarted
deployment.apps/grampsweb-celery restarted
{% endhighlight %}

Tail the logs of `Celery` and wait for it to become ready:
{% highlight shell %}
kubectl logs deployment/grampsweb-celery
...
<timestamp> celery@grampsweb-celery-fbf679f5c-m96md ready.
{% endhighlight %}

## Verify

If we go to our `Grampsweb` in a web browser, we should now see the `first run` page.
To verify that `Grampsweb` is correctly using the `PostgreSQL` database for users, we can create an `Admin user`, and then log into the `PostgreSQL instance` and check that a corresponding row is added in the `users` table in the `users` database.

After verifying that `users` behaves as expected, we can `add` a `Person` in the family tree. Once added we can verify that records are added to the `documents` table in the `search` database.

# Cleanup
## Remove volume mounts:
We can now remove the `gramps-users` and the `gramps-index` volumes from both the `Grampsweb` and the `Celery` `Deployments`
{% highlight yaml %}
...
            - mountPath: /app/users
              name: gramps-users
            - mountPath: /app/indexdir
              name: gramps-index
...

        - name: gramps-secret
          persistentVolumeClaim:
            claimName: gramps-users
        - name: gramps-secret
          persistentVolumeClaim:
            claimName: gramps-index
...
{% endhighlight %}

[grampsweb-deployment-with-database.yaml]({% link /assets/files/grampsweb/grampsweb-deployment-with-database.yaml %})

[grampsweb-celery-deployment-with-database.yaml]({% link /assets/files/grampsweb/grampsweb-celery-deployment-with-database.yaml %})

## Deploy changes to grampsweb and celery

{% highlight yaml %}
kubectl apply -n gramps -f grampsweb-deployment.yaml -f grampsweb-celery-deployment.yaml
deployment.apps/grampsweb configured
deployment.apps/grampsweb-celery configured
{% endhighlight %}

this will restart the `Deployments`. To verify that the restart completed, we can tail the log of `Celery` and see that it becomes ready:
{% highlight shell %}
kubectl logs deployment/grampsweb-celery
...
<timestamp> celery@grampsweb-celery-fbf679f5c-m96md ready.
{% endhighlight %}

## Delete Volumes
When everything is confirmed working, we no longer need the the PersistentVolumeClaim for gramps-users and gramps-index:

{% highlight shell %}
kubectl delete PersistentVolumeClaim gramps-users gramps-index
persistentvolumeclaim "gramps-users" deleted
persistentvolumeclaim "gramps-index" deleted
{% endhighlight %}

# Profit!
With these changes we now have all our `users` stored in an external `database` that is easy to `back up` and `restore` if needed. We have also removed the dependencies on `shared filesystem` access, something that is an `antipattern` in the world of (mostly) stateless microservices.
In [the next part]({% link _posts/2025-05-17-Grampsweb-on-kubernetes-part5.markdown %}) we will look at replacing the default, file based, `SQLight` database for the `family tree`
