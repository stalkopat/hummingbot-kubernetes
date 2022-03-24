Unless mentioned otherwise all Passwords set in the Configs will be the following string: ChangeM3Pl34se

# Setup
Installing Kubernetes Tools locally is fairly straightforward, see: https://kubernetes.io/docs/tasks/tools/ for more information
Setting up a Kubernets Cluster from scratch is trickier, but luckily there is a lot of managed Providers that can do the
setup for you, for example Linode, DigitalOcean, Vultr, Google Cloud and many many more. The vast majority of those offerings
will be plenty enough to run hummingbot instances on and they don't differ much in terms of core functionality.

# Kubernetes
Kubernetes is a Container Orchestration tool. With it you can abstract away the underlying infrastructure from your containers.
Kubernetes also allows you to do a whole bunch of fun things like automatic failover, rolling updates and much more.
Most of those features don't really work for hummingbot though, since each container represents a single instance.
But there is still a lot of quality of life improvements:
    - Quick mass deployment of Hummingbot instances
    - No more fiddling with start / stop commands and config files
    - No longer bound to a single Virtual Machine / VPS
    - Easy overview of current instances
    - Effortlessly scale both horizontally and vertically

In Kubernetes a Container itself is stateless, it can be destroyed and recreated at will by both the cluster itself and by yourself.
Due to that the configurations are the only differing factor between two containers, everything else can be switched up at will.
There are persistent volumes one could attach for keep logs or different data. I personally haven't used anything in that direction
but its an option if one has a requirement to keep logs for some reason.

# Kubernetes vs Docker
Now you might think that they both fullfill the same purpose, but there is a distinct difference between them, Docker is to an
Application what Kubernetes is to your Infrastructure. You still have to care about the underlying infrastructure when using Docker
on individual Virtual Machines. 
Here is an example deployment of 3 Hummingbot Instances via both source and via docker:
![image](https://user-images.githubusercontent.com/25117613/159880909-7f5e27b4-4305-4246-9759-4365e5511b88.png)

Kubernetes abstracts the connection between Containers and the hardware away:
![image](https://user-images.githubusercontent.com/25117613/159881182-bb8e203f-e3aa-4f69-89f0-ca9f970e709d.png)

You don't have to care about the underlying hardware anymore, it can be created, destroyed and changed at will, 
without you having to do anything to accomodate your hummingbot instances

# Hummingbot inside of Kubernetes
There is a lot of ways to deploy a (docker) container inside of Kubernetes. Luckily for us Hummingbot is self contained and
only needs connection to the exchanges it market makes on, because of that we can skip the more advanced ressources like DeamonSet 
and ReplicaSet for example. I personally have had a good experience using StatefulSets. Each hummingbot instance is 
represented through 3 different ressources inside of my workflow:

1. A configmap -> a configmap is a key value store used to save any configurations
2. A ServiceAccount -> probably not neccessary but allows one to have different accounts per instance
3. A StatefulSet -> Contains the Hummingbot Container and logic to plug the configurations in

The configmap is mounted as a volume and then copied into the local filesystem of the container so hummingbot can read the
configuration .yml files. The "cp -r /readonly-conf/. /conf ;" start parameter is necessary because configmap volumes are always
readonly which causes issues for hummingbot.

# Deployment of an Instance
After setting up your kubectl with the appropriate configuration files inside ~/.kube/ you are ready to deploy a hummingbot instance.
There is an example file in this repository that you can use to create a BTC-USDT Pure Market Making bot on ascendex_paper_trade.

To deploy the configuration file including the 3 Ressources it contains you can use the following command:

    kubectl apply -f ./Example-StatefulSet-BTC-USDT.yaml

using

    kubectl get pods

you can watch the current status of your pod (A pod is a wrapper around you container):

NAME         READY   STATUS              RESTARTS   AGE

btc-usdt-0   0/1     ContainerCreating   0          22s

The first time you deploy a Pod it will take a bit of time because it has to pull the docker image 
from coinalphas dockerhub, subsequent deployments should run through quicker.
After a bit of time you will see that the status has changed to "Running" this indicates that the hummingbot instance is now live:

NAME         READY   STATUS    RESTARTS   AGE

btc-usdt-0   1/1     Running   0          2m2s

Now of course you won't know what exactly it is doing inside of the pod, since you cant access the user interface.
You can access the current output of the instance via

    kubectl logs btc-usdt-0

I recommend doing so inside of a visual studio terminal, hummingbot updates the console screen via control characters 
and in my experience only powershell inside of the vscode terminal is able to properly parse them and show you the current output.

# FAQ
1. Can kubernetes run servers at multiple, different service providers like Lansol, Contabo, Linode, Digital Ocean, etc?

Yes, kubernetes can run a cluster across multiple providers, but its rather complicated unless you get another 
provider to do the managing for you, take a look at https://cloud.google.com/anthos

2. Is there limitation to no. of bots it can scale up to?

Realistically there isn't any limitation besides how deep your pockets are, technically you can only go up to 110 hummingbot 
instances per underlying node and up to 300'000 total hummingbot instances but you won't realistically reach that size.

3. Does running kubernetes needs a lot of maintenance, or deeper know-how?

Kubernetes is really low maintenance unless you roll your own, when using a managed provider you will have automatic updates of the cluster (via rolling update, pods get moved from one node to the other(s). That one node shuts down, updates itself, then comes back online, pods get moved to the original node and the second node updates). Deeper know how isn't really necessary because of hummingbots "simplistic" nature of not being interconnected with other pods / services.
