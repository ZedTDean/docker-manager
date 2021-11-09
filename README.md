![Logo](https://www.gurobi.com/wp-content/uploads/2018/12/logo-final.png "Gurobi Optimization")
# Quick reference
Maintained by: [Gurobi Optimization](https://www.gurobi.com)

Where to get help: [Gurobi Support](https://www.gurobi.com/support/), [Gurobi Documentation](https://www.gurobi.com/documentation/)

# Supported tags and respective Dockerfile links

* [9.5.0, latest](https://github.com/Gurobi/docker-manager/blob/master/9.5.0/Dockerfile)
* [9.5.0](https://github.com/Gurobi/docker-manager/blob/master/9.5.0/Dockerfile)

When building a production application, we recommend using an explicit version number instead of the `latest` tag.
This way, you are in control of the upgrade process of your application.

# Quick reference (cont.)

Supported architectures: linux amd64

Published image artifact details: https://github.com/Gurobi/docker-manager

Gurobi images:
- [gurobi/optimizer](https://hub.docker.com/r/gurobi/optimizer): Gurobi Optimizer (full distribution)
- [gurobi/python](https://hub.docker.com/r/gurobi/python): Gurobi Optimizer (Python API only)
- [gurobi/python-example](https://hub.docker.com/r/gurobi/python-example): Gurobi Optimizer example in Python with a WLS license
- [gurobi/modeling-examples](https://hub.docker.com/r/gurobi/modeling-examples): Optimization modeling examples (distributed as Jupyter Notebooks)
- [gurobi/compute](https://hub.docker.com/r/gurobi/compute): Gurobi Compute Server
- [gurobi/manager](https://hub.docker.com/r/gurobi/manager): Gurobi Cluster Manager

# What is `gurobi/manager`?
The Gurobi Optimizer is the fastest and most powerful mathematical programming solver available 
for your LP, QP and MIP (MILP, MIQP, and MIQCP) problems. 
More info at the [Gurobi Website](https://www.gurobi.com/products/gurobi-optimizer/).

The Gurobi Cluster Manager is a new server component that helps to manage your Compute
Server. It provides better security through user authentication and API keys. 
It also expands the capabilities of Compute Server cluster nodes with unified management of 
interactive and non-interactive optimization tasks. 
[Gurobi Compute Server Documentation](https://www.gurobi.com/documentation/current/remoteservices/index.html)
 
## Getting a Gurobi license

The [Web License Service](https://www.gurobi.com/web-license-service/) (WLS) is a new Gurobi licensing service 
for containerized environments (Docker, Kubernetes...). Gurobi components can automatically request and renew license tokens to 
the WLS servers available in several regions worldwide. WLS only requires that your container has access to the 
Internet. Commercial users can request an evaluation and academic users can request a free license.
Please register to access the [Web License Manager](https://license.gurobi.com) and read the
[documentation](https://license.gurobi.com/manager/doc/overview)

Note that other standard license types (NODE, Academic) do not work with containers.
Please contact your sales representative at [sales@gurobi.com](mailto:sales@gurobi.com) to discuss licensing options. 

## Using the client license

The Cluster Manager itself does not need a license, but the Compute Server nodes will, and there are two options:
* Mounting the client license file:
You can store connection parameters in a client license file (typically called `gurobi.lic`) 
and mount it to the container. 
This option provides a simple approach for testing with Docker.
When using Kubernetes, the license file can be stored as a secret and mounted in the container.

* Setting parameters through environment variables for a WLS license: GRB_WLSACCESSID, GRB_WLSSECRET, and GRB_LICENSEID.
These variables are used to pass the access ID, the secret, and the license ID respectively.

We do not recommend adding the license file to the Docker image itself. It is not a flexible 
solution as you may not reuse the same image with different settings. More importantly, it is not secure
as some license files need to contain credentials in the form of API keys that should remain private.

# How to use this image?
## Using Docker

The following command starts a Cluster Manager instance and connects to a MongoDB database
`$ docker run gurobi/manager --database=MONGO_DB_URL`

Then you can go to http://localhost:61080/manager and log in using the default credentials:
```
    standard user: gurobi / pass
    administrator: admin / admin
    system administrator: sysadmin / cluster

```

However, this is for testing only because without at least one Compute Server node, you
will not be able to submit optimization jobs. This is why using Docker Compose or Kubernetes
is necessary as the deployment scripts will start all the required components (see below).

## Using Docker Compose
Example `docker-compose.yml` for a cluster manager:

```
version: '3.1'
services:

  mongodb:
    image: mongo:latest
    restart: always
    volumes:
      - mongodb:/data/db
    networks:
      - back-tier

  manager:
    image: gurobi/manager:latest
    restart: always
    depends_on:
      - "mongodb"
    ports:
      - "61080:61080"
    command: --database=mongodb://mongodb:27017
    networks:
      - back-tier
  
  compute:
    image: gurobi/compute:latest
    restart: always
    depends_on:
      - manager
    command: --manager=http://manager:61080
    networks:
      - back-tier
    volumes:
      - ./gurobi.lic:/opt/gurobi/gurobi.lic:ro

volumes:
  mongodb:

networks:
  back-tier:

```

Run `docker-compose up` from the directory where the `docker-compose.yml` file was saved to download and start all the containers.

Then you can go to http://localhost:61080/manager and log in using the default credentials:
```
    standard user: gurobi / pass
    administrator: admin / admin
    system administrator: sysadmin / cluster

```

Note that you can scale `gurobi/compute` instances using the [scale](https://docs.docker.com/compose/reference/scale/) flag:
```
$ docker-compose up --scale compute=2

```

## Using Kubernetes

If you want to mount a specific license with Kubernetes, it can be done using a secret. 

```
kubectl create secret generic gurobi-lic --from-file="gurobi.lic=$PWD/gurobi.lic"
```

Then you can start multiple pods for the Cluster Manager, the Mongo Database, and the Compute Server nodes.
A simple deployment file is provided as an 
[example](https://github.com/Gurobi/docker-manager/blob/master/9.1.2/k8s.yaml).
To keep the demonstration simple, this deployment file will not persist the database.
If you wish to do so, please refer to the [MongoDB documentation](https://www.mongodb.com/kubernetes)
or a hosted solution (for example [MongoDB Atlas](https://www.mongodb.com/cloud/atlas)).

```
kubectl apply -f k8s.yaml
```

To check the deployment, you can list the created pods:
```
% kubectl get pods 
NAME                                       READY   STATUS    RESTARTS   AGE
gurobi-compute-674f8dcbf4-v27nn            1/1     Running   0          8m16s
gurobi-compute-674f8dcbf4-x9kff            1/1     Running   0          8m23s
gurobi-manager-79dcbf5b74-c9dfv            1/1     Running   0          8m23s
gurobi-manager-79dcbf5b74-wcqhj            1/1     Running   0          8m17s
```

Then you can access the logs of one instance of the Cluster Manager:
```
% kubectl logs gurobi-manager-79dcbf5b74-c9dfv
2021-04-22T02:57:28Z - info  : Gurobi Cluster Manager starting...
2021-04-22T02:57:28Z - info  : Platform is linux
2021-04-22T02:57:28Z - info  : Version is 9.1.2 (build v9.1.2rc0)
2021-04-22T02:57:28Z - info  : Running in a Docker container 92b6ab343ae877b932aa98155b202bdc4b8bd36d66c4d94f38cf85e7e46ade0d
2021-04-22T02:57:28Z - info  : Connecting to database grb_rsm on 10.107.136.76:27017...
2021-04-22T02:57:28Z - info  : Connected to database grb_rsm (version 4.2.13, host gurobi-mongo-7cb74f9b7-c829s)
2021-04-22T02:57:28Z - info  : Creating proxy with MaxIdleConns=200 MaxIdleConnsPerHost=32 IdleConnTimeout=130
2021-04-22T02:57:28Z - info  : Public root is /opt/gurobi_server/linux64/resources/grb_rsm/public
2021-04-22T02:57:28Z - info  : Starting Cluster Manager server (HTTP) on port 61080...
```

As well as the logs of one of the Compute Server nodes:
```
% kubectl logs gurobi-compute-674f8dcbf4-v27nn
2021-04-22T02:57:32Z - info  : Gurobi Remote Services starting...
2021-04-22T02:57:32Z - info  : Platform is linux
2021-04-22T02:57:32Z - info  : Version is 9.1.2 (build v9.1.2rc0)
2021-04-22T02:57:32Z - info  : Running in a Docker container 293626443b09bb0d19a3d97e918ea0d11d2ae0cb92be60ae944c92848735471e
2021-04-22T02:57:32Z - info  : Variable GRB_LICENSE_FILE is not set
2021-04-22T02:57:32Z - info  : Using license file /opt/gurobi/gurobi.lic
2021-04-22T02:57:32Z - info  : Server starting WLS license
2021-04-22T02:57:32Z - info  : Node address is 10.1.0.36:61000
2021-04-22T02:57:32Z - info  : Node FQN is gurobi-compute-674f8dcbf4-v27nn
2021-04-22T02:57:32Z - info  : Node has 6 cores
2021-04-22T02:57:32Z - info  : Using data directory /opt/gurobi_server/linux64/bin/data
2021-04-22T02:57:32Z - info  : Data store created
2021-04-22T02:57:32Z - info  : Node ID is f5da76ef-40ca-4b26-9bbf-43b9a0a1b5c8
2021-04-22T02:57:32Z - info  : Available runtimes: [8.0.0 8.0.1 8.1.0 8.1.1 9.0.0 9.0.1 9.0.2 9.0.3 9.1.0 9.1.1 9.1.2]
2021-04-22T02:57:32Z - info  : Public root is /opt/gurobi_server/linux64/resources/grb_rs/public
2021-04-22T02:57:32Z - info  : Starting API server (HTTP) on port 61000...
2021-04-22T02:57:32Z - info  : Accepting worker registration on port 43209...
2021-04-22T02:57:32Z - info  : Joining cluster from manager
2021-04-22T02:57:33Z - info  : *** WLS license - registered to gurobi@gurobi.com ***
```

Finally, you can retrieve the load balancer ingress of the ``gurobi-manager`` service
and open your browser to the exposed url and port. If you have applied the deployment file
to a local environment, you can go to ``http://localhost:61080/manager`` and log in using the 
default credentials:
```
    standard user: gurobi / pass
    administrator: admin / admin
    system administrator: sysadmin / cluster

```

Gurobi Compute Server will run the optimization jobs which are CPU-intensive tasks. To get better 
performance, we recommend deploying the Compute Server pods on dedicated Kubernetes worker nodes with 
high-end CPUs, adequate memory, and avoid resource contention: 
- By using a [daemon set](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/), 
  we run one and only one Gurobi Compute Server per worker node as an agent. Also, if you scale up or down
  the number of nodes in your node group, it will automatically adjust the number of Compute Server nodes.
- Then, the node selector has a label ``app: compute`` to indicate that the daemon will only run on the 
  dedicated worker nodes.
- Finally, by defining a [taint](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) 
  ``app:compute`` on the selected worker nodes, and applying the tolerations 
   to the daemon set, we make sure no other pods will run on these nodes, unless they have the required toleration.

A simple deployment file is provided as an
[example](https://github.com/Gurobi/docker-manager/blob/master/9.1.2/daemonset.yaml).

# License

By downloading and using this image, you agree with the 
[End-User License Agreement](https://www.gurobi.com/EULA) for the Gurobi software contained in this image.

As with all Docker images, these likely also contain other software which may be under other 
licenses (such as Bash, etc from the base distribution, along with any direct or indirect 
dependencies of the primary software being contained).

As for any pre-built image usage, it is the image user's responsibility to ensure that any use 
of this image complies with any relevant licenses for all software contained within.

