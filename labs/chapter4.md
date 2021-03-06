## LAB 4: Orchestrated deployment of a decomposed application (Langdon)

In this lab we introduce how to orchestrate a multi-container application in Kubernetes.

Expected completion: 40-60 minutes

Let's start with a little experimentation. I am sure you are all excited about your new blog site! And, now that it is getting super popular with 1000s of views per day, you are starting to worry about uptime.

However, before we get started, let's do a bit of cleanup from the prior labs. Let's delete the old containers (the ```-f``` forces the removal even if the container is running:
```
docker rm -f wordpress mariadb
```  

So, let's see what will happen. Launch the site:
```
docker run -d -p 3306:3306 -e DBUSER=user -e DBPASS=mypassword -e DBNAME=mydb --name mariadb mariadb
docker run -d -p 80:80 --link mariadb:db --name wordpress wordpress
```

Take a look at the site in your web browswer on your machine using **http://ip:port**

Now, let's see what happens when we kick over the database. However, for a later experiment, let's grab the container-id right before you do it. 

```
OLD_CONTAINER_ID=$(docker inspect --format '{{ .Id }}' mariadb)
docker stop mariadb
```

Take a look at the site in your web browswer now. And, imagine, explosions! *making sound effects will be much appreciated by your lab mates.*
```
web browswer -> **http://ip:port**
```

Now, what is neat about a container system, assuming your web application can handle it, is we can bring it right back up, with no loss of data.
```
docker start mariadb
```

OK, now, let's compare the old container id and the new one. 
```
NEW_CONTAINER_ID=$(docker inspect --format '{{ .Id }}' mariadb)
echo -e "$OLD_CONTAINER_ID\n$NEW_CONTAINER_ID"
```

Hmmm. Well, that is cool, they are exactly the same. OK, so all in all, about what you would expect for a web server and a database running on VMs, but a whole lot faster. However, what if we could automate the recovery? Or, in buzzword terms, "ensure the service remains up"? Enter Kubernetes. And, so you are up on the lingo, sometimes "kube" or "k8s".

### Pod Creation

Let's get started by talking about a pod. A pod is a set of containers that provide one "service." The sense here is that the pods need to be co-located on a host and need to be spawned and re-spawned together.

Let's make a pod for mariadb. Open a file called mariadb-pod.yaml.
```
mkdir -p ~/workspace/mariadb/kubernetes
vi ~/workspace/mariadb/kubernetes/mariadb-pod.yaml
```
In that file, let's put in the pod identification information:
```
apiVersion: v1beta3
kind: Pod
metadata:
  labels:
    name: mariadb
  name: mariadb
spec:
  containers:
```

We specified the version of the Kubernetes api, the name of this pod (aka ```name```), the ```kind``` of Kubernetes thing this is, and a ```label``` which lets other Kubernetes things find this one.

Generally speaking, this is the content you can copy and paste between pods, aside from the names and labels.

Now, let's add the custom information regarding this particular container. To start, we will add the most basic information. Please be sure to replace "YOUR_LAB_DEV_MACHINE" with the name of your machine. Replace the ```containers:```

```
  containers:
  - capabilities: {}
    env:
    image: YOUR_LAB_DEV_MACHINE:5000/mariadb
    name: mariadb
    ports:
    - containerPort: 3306
      protocol: TCP
    resources:
      limits:
        cpu: 100m
```
Here we set the ```name``` of the container; remember we can have more than one in a pod. We also set the ```image``` to pull, in other words, the container image that should be used and the registry to get it from. We can also set limitations here like cpu cap and exposed ports.

Lastly, we need to configure the environment variables that need to be fed from the host environment to the container. Replace ```env:``` with:
```
    env:
    - name: DBUSER
      value: user
    - name: DBPASS
      value: mypassword
    - name: DBNAME
      value: mydb
```

OK, now we are all done, and should have a file that looks like:
```
apiVersion: v1beta3
kind: Pod
metadata:
  labels:
    name: mariadb
  name: mariadb
spec:
  containers:
  - capabilities: {}
    env:
    - name: DBUSER
      value: user
    - name: DBPASS
      value: mypassword
    - name: DBNAME
      value: mydb
    image: YOUR_LAB_DEV_MACHINE:5000/mariadb
    name: mariadb
    ports:
    - containerPort: 3306
      protocol: TCP
    resources:
      limits:
        cpu: 100m
```
Be sure to replace "YOUR_LAB_DEV_MACHINE" with the name of your machine if you jumped to this point in the lab.

Our wordpress container is much less complex, so let's do that pod next.

```
mkdir -p ~/workspace/wordpress/kubernetes
vi ~/workspace/wordpress/kubernetes/wordpress-pod.yaml
```

```
apiVersion: v1beta3
kind: Pod
metadata:
  labels:
    name: wpfrontend
  name: wordpress
spec:
  containers:
  - env:
    - name: DB_ENV_DBUSER
      value: user
    - name: DB_ENV_DBPASS
      value: mypassword
    - name: DB_ENV_DBNAME
      value: mydb
    image: YOUR_LAB_DEV_MACHINE:5000/wordpress
    name: wordpress
    ports:
    - containerPort: 80
      protocol: TCP
```

A couple things to notice about this file. Obviously, we change all the appropriate names to reflect "wordpress" but, largely, it is the same as the mariadb pod file. We also use the environment variables that are specified by the wordpress container, although they need to get the same values as the ones in the mariadb pod. Lastly, just to show you aren't bound to the image or pod names, we also changed the ```labels``` value to "wpfronted". Be sure to replace "YOUR_LAB_DEV_MACHINE" with the name of your machine as with the maria-pod.

Ok, so, lets launch our pods and make sure they come up correctly. In order to do this, we need to introduce the ```kubectl``` command which is what drives Kubernetes. Generally, speaking, the format of ```kubectl``` commands is ```kubetctl <operation> <kind>```. Where ```<operation>``` is something like ```create```, ```get```, ```remove```, etc. and ```kind``` is the ```kind``` from the pod files.

```
kubectl create -f ~/workspace/mariadb/kubernetes/mariadb-pod.yaml
kubectl create -f ~/workspace/wordpress/kubernetes/wordpress-pod.yaml
```
Now, I know i just said, ```kind``` is a parameter, but, as this is a create statement, it looks in the ```-f``` file for the ```kind```.

Ok, let's see if they came up:
```
kubectl get pods
```

Which should output two pods, one called ```mariadb``` and one called ```wordpress```.

Ok, now let's kill them off so we can introduce the services that will let them more dynamically find each other.
```
kubectl delete pod mariadb
kubectl delete pod wordpress
```
**Note** you used the "singular" form here on the ```kind```, which, for delete, is required and requires a "name". However, you can, usually, use them interchangably depending on the kind of information you want.

### Service Creation
Now we want to create Kubernetes Services for our pods so that Kubernetes can introduce a layer of indirection between the pods. 

Let's start with mariadb. Open up a service file:
```
vi ~/workspace/mariadb/kubernetes/mariadb-service.yaml
```

and insert the following content:

```
apiVersion: v1beta3
kind: Service
metadata:
  labels:
    name: mariadb
  name: mariadb
spec:
  ports:
  - port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    name: mariadb
```
As you can probably tell, there isn't really anything new here. However, you need to make sure the ```kind``` is of type ```Service``` and that the ```selector``` matches at least one of the ```labels``` from the pod file. The ```selector``` is how the service finds the pod that provides its functionality.

OK, now let's move on to the wordpress service. Open up a new service file:
```
vi ~/workspace/wordpress/kubernetes/wordpress-service.yaml
```

and insert:
```
apiVersion: v1beta3
kind: Service
metadata:
  labels:
    name: wpfrontend
  name: wpfrontend
spec:
  createExternalLoadBalancer: true
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    name: wpfrontend
```
So, here you may notice, there is no reference to wordpress at all. In fact, we might even want to name the file wpfrontend-service.yaml to make it clearer that, in fact, we could have any pod that provides "wordpress capabilities". However, for a lab like this, I thought it would be confusing. 

An even better example might have been if we had made the mariadb-service just a "db" service and then, the pod could be mariadb, mysql, sqlite, anything really, that can support SQL the way wordpress expects it to. In order to do that, we would just have to add a ```label``` to the ```mariadb-pod.yaml``` called "db" and a ```selector``` in the ```mariadb-service.yaml``` (although, an even better name might be ```db-service.yaml```) called ```db```. Feel free to experiment with that at the end of this lab if you have time.

Now let's get things going. Start mariadb:
```
kubectl create -f ~/workspace/mariadb/kubernetes/mariadb-pod.yaml
kubectl create -f ~/workspace/mariadb/kubernetes/mariadb-service.yaml
```

Now let's start wordpress.
```
kubectl create -f ~/workspace/wordpress/kubernetes/wordpress-pod.yaml
kubectl create -f ~/workspace/wordpress/kubernetes/wordpress-service.yaml
```

OK, now let's make sure everything came up correctly:
```
kubectl get pods
kubectl get services
```
**Note** these may take a while to get to a ```RUNNING``` state as it pulls the image from the registry, spin up the containers, do the kubernetes magic, etc. 

Eventually, you should see:
```
POD         IP            CONTAINER(S)   IMAGE(S)                         HOST                  LABELS            STATUS    CREATED
mariadb     172.17.0.17   mariadb        summit-rhel-dev:5000/mariadb     127.0.0.1/127.0.0.1   name=mariadb      Running   11 hours
wordpress   172.17.0.18   wordpress      summit-rhel-dev:5000/wordpress   127.0.0.1/127.0.0.1   name=wpfrontend   Running   11 hours
```

```
NAME            LABELS                                    SELECTOR          IP               PORT(S)
kubernetes      component=apiserver,provider=kubernetes   <none>            10.254.0.2       443/TCP
kubernetes-ro   component=apiserver,provider=kubernetes   <none>            10.254.0.1       80/TCP
mariadb         name=mariadb                              name=mariadb      10.254.252.125   3306/TCP
wpfrontend      name=wpfrontend                           name=wpfrontend   10.254.127.139   80/TCP
```

Seemed awfully manual and ordered up there, didn't it? Just wait til Lab5 where we make it a lot less painful!

Now that we are satisfied that our containers and Kubernetes definitions work, let's try deploying it to a remote server.

First, we have to add the remote cluster to our local configuration. However, before we do that, let's take a look at what we have already. Also, notice that the ``kubectl config``` follows the <noun> <verb> model. In other words, ```kubectl``` <noun>=```config``` <verb>=```view``` 
```
kubectl config view
``` 

Not much right? If you notice, we don't even have any information about the current context. In order to avoid losing our local connection, why don't we set up the local machine as a cluster first, before we add the remote. However, in order for the configuration to work correctly, we need to touch the config file first.
```
mkdir ~/.kube
touch ~/.kube/.kubeconfig
```

First we create the cluster (after each step, I recommend you take a look at the current config with a ```view```):
```
kubectl config set-cluster local --server=http://localhost:8080
kubectl config view
```

Then we add it to a context:
```
kubectl config set-context local-context --cluster=local
kubectl config view
```

Now we switch to that context:
```
kubectl config use-context local-context
kubectl config view
```

Strictly speaking, a lot of the above is not necessary, however, it is good to get in to the habit of using "contexts" then when you are using ```kubectl``` with properly configured security and the like, you will run in to less "mysterious" headaches trying to figure out why you can't deploy.

Now, lets test it out.
```
kubectl get pods
kubectl get services
```

Did you get your pods and services back? If not, you should check your config. Your ```config view``` result should look like this:
```
kubectl config view
```
Result:

```
apiVersion: v1
clusters:
- cluster:
    server: http://localhost:8080
  name: local
contexts:
- context:
    cluster: local
    user: ""
  name: local-context
current-context: local-context
kind: Config
preferences: {}
users: []
```

All right, let's switch to the remote, don't forget to replace YOUR_LAB_DEPLOY_MACHINE with the correct name:
```
kubectl config set-cluster remote --server=http://YOUR_LAB_DEPLOY_MACHINE:8080
kubectl config set-context remote-context --cluster=remote
kubectl config use-context remote-context
kubectl config view
```
You should now have ```current-context: remote-context```. Now, let's prove we are talking to the remote:
```
kubectl get pods
kubectl get services
```
Nothing there, right? Ok, so let's start the bits up on the remote, deployment server: 
```
kubectl create -f ~/workspace/mariadb/kubernetes/mariadb-pod.yaml
kubectl create -f ~/workspace/mariadb/kubernetes/mariadb-service.yaml
kubectl create -f ~/workspace/wordpress/kubernetes/wordpress-pod.yaml
kubectl create -f ~/workspace/wordpress/kubernetes/wordpress-service.yaml
```

Now we should see similar results as our local machine from:
```
kubectl get pods
kubectl get services
```

Now we can check to make sure the site is running. However, first we need the IP for it.
```
kubectl get endpoints
```
Which should give you a result like:
```
NAME            ENDPOINTS
kubernetes      192.168.121.102:6443
kubernetes-ro   192.168.121.102:7080
mariadb         172.17.0.17:3306
wpfrontend      172.17.0.18:80
```

In order for this to work, we need to create a tunnel in to the deploy machine to proxy connections to the wordpress server. In order to do that we use the "wpfrontend endpoint" from above and insert it in to the following command which needs to be executed on the deploy machine:
```
ssh root@summit_rhel_deploy_target
ssh -L *:9080:<WPFRONTEND_ENDPOINT>:80 root@localhost
```

We need to create a tunnel because in the lab environment we have to use subnets on the VMs for Kubernetes to work. Now in your browser type in:
```
http://summit-rhel-deploy-target:9080
```

Ok, now you can move on to lab5, where Aaron will show you how to create an application much more easily.
