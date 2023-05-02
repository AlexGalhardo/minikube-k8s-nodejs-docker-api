k8s - Deploying a NodeJS app in Minikube (Local development)

This blog will walk you through the steps of how to deploy a NodeJS application in MiniKube. We will be using a local docker image without relying on a Docker registry.

At this stage, I’m assuming you have a general idea of what Kubernetes is. If not <https://kubernetes.io/> is a great place to start.

Before starting the tutorial, make sure you have the following installed on your machine (Assuming Docker and Node are pre-installed)

minikube: <https://minikube.sigs.k8s.io/docs/start/>
— Minikube is a tool to learn and develop Kubernetes locally

kubectl: <https://kubernetes.io/docs/tasks/tools/install-kubectl/>
— Kubectl is a command-line tool that allows you to control the Kubernetes cluster.

Note: Read more on Kubernetes components <https://kubernetes.io/docs/concepts/overview/components/>

With all this in hand, let’s start.

First, let’s create a new folder to add our source codes. I will name this as kubernetes-test and inside that let’s have anode-app folder to place our NodeJS app. (Feel free to name them as you please)

NodeJS app
In your favorite terminal navigate to node-app folder and run

npm init -y # Generate package.json
npm i express # Install express
touch index.js # Create a new index.js file
(Depending on your operating system, alter thetouch command to create a new file)

At this stage, you are free to create any NodeJS app you want. The app I will be building is a web server that returns you a set of Star Wars character names.
I will be using the node module: <https://www.npmjs.com/package/unique-names-generator> to get random Star Wars character names. If you are following along, run the below command.

npm i unique-names-generator
index.js will look like

```js
const express = require('express');
const { uniqueNamesGenerator, starWars } = require('unique-names-generator');
const app = express();
// Get maximum character from ENVs else return 5 character
const MAX_STAR_WARS_CHARACTERS = process.env.MAX_STAR_WARS_CHARACTERS || 5;
const config = {
  dictionaries: [starWars]
}
// Get the character name array
const getStarWarsCharacters = () => {
 const characterNames = [];
for (let i = 1; i <= MAX_STAR_WARS_CHARACTERS; i += 1) {
  characterNames.push(uniqueNamesGenerator(config));
 }
 return characterNames;
};
app.get('/', (req, res) => {
 res.json(getStarWarsCharacters());
});
app.listen(3000, () => {
 console.log('Server started on port 3000');
});
```

Few things to note here

The maximum number of characters returned will be governed by an ENV (if MAX_STAR_WARS_CHARACTERS is not defined return 5 characters)
Characters will be returned when a GET is requested at the root path /
In the package.json file, under the scripts object, add the following.

```
"scripts": {
    "start": "node index.js"
 }
 ```

Now let’s run the server to check if our app works as intended. Type
npm start and you should get the following log.

Server started on port 3000
Then on a separate terminal run,

```
curl <http://localhost:3000>
```

```
["Bail Prestor Organa","Palpatine","Quarsh Panaka","Boba Fett","Padmé Amidala"]
```

You should get 5 Star Wars characters.
Stop the server and run,

export MAX_STAR_WARS_CHARACTERS=3
npm start
Now you should only get 3 Star Wars characters.
Now we know our server is working fine.

### Dockerfile

In the same node-app directory, create aDockerfile and place the following content.

```
FROM node:12-alpine
WORKDIR /app
COPY package*.json ./
RUN npm i
COPY index.js index.js
CMD ["npm", "start"]
```

If you are wondering why there are 2 COPY statements, it is to take advantage of caching for npm i .

Now let’s build and run the docker image to see if dockerization works as expected. Run,

docker build -t starwars-node .
Now if you run docker images you should see the newly created image starwars-node and now run,

```
docker run -d -p 3000:3000 starwars-node

# 8b55e8c1c32d39a5b39ab38cb41c80e6cc05843a0baef625664939435183f51b
```

You should get the id of the newly created container.
Run the samecurl command and see if you can see your favorite Star Wars character names.
Note: by adding --env to docker run command you can change the number of characters returned.

Kubernetes

Up to this point, we have been doing stuff that supports the main content.
From this point onward let’s work with minikube and kubectl .

First, we need to start the Kubernetes cluster. We can do this by running.

minikube start
If all successful, you will see the following message at the end of the console.

Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
We are ready to use kubectl to deploy our application to the cluster.

Deployment

In Kubernetes aPod is an instance of a running container. Node or a Worker machine in the Kubernetes cluster will contain one or many pods. In order to create a pod(s), we can use a deployment object defined via a .yaml file.

Navigate to kubernetes-test folder and create sw-deployment.yaml file with the following content.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: star-wars-deployment
spec:
  selector:
    matchLabels:
      app: star-wars
  replicas: 2
  template:
    metadata:
      labels:
        app: star-wars
    spec:
      containers:
        - name: star-wars
          image: starwars-node
          ports:
            - containerPort: 3000
```

kubectl as described earlier used to control the k8 cluster. This is done by using something called API Server.

Fields in the .yaml file have the following meanings.
apiVersion indicated which API version used to create this object.
kind is the type of Object to be created.
metadata.name is the name given to the created object.
spec.replicas is the number of replicated Pods . We have specified 2.
spec.selector is a way for deployments to find the Pods . We have given app: star-wars label.
spec.template with its subfields is a way to specify the pod and container information.
template.labels.app specified the label of the pod, here it is star-wars template.spec.containers is an array that allows you to specify which image(s) needed by a pod and ports it will be exposing. We have specified the image we have created starwars-node image which exposes port 3000.

Note: Read deployment for more information.

Now to create Pods run the following command.

```
kubectl apply -f sw-deployment.yaml
```

# deployment.apps/star-wars-deployment created

Now if you run the below command. You should see two pods.

```
kubectl get pod
```

If you look really close atSTATUS of each pod, they might be something like ErrImagePull or ImagePullBackOff and if you run one of the k8’s debug commands like logs or describe pod .

```
kubectl logs <pod_name>
```

# Error from server (BadRequest): container "star-wars" in pod "<pod_name>" is waiting to start: trying and failing to pull image

It is failing to pull the image. Can you spot the issue here?

As you might have guessed, the docker daemon in the cluster (MiniKube runs in a separate Docker container or a VM) is trying to find an image that does not exist. We only created the image in the above steps for the logged-in user. So we need to create the same image inside the Kubernetes cluster.

Go to node-app folder again and run the following commands.

```
eval $(minikube docker-env)
docker build -t starwars-node .
```

# Additionally you can check to see if starwars-node Docker image is in minikube by

```
minikube ssh
docker@minikube:~$ dokcer images
```

# You should see starward-node docker image

In the sw.deployment.yaml add the following line

```
ports:
- containerPort: 3000
imagePullPolicy: Never # Image should not be pulled
```

Note: refer blogpost for additional information on this issue.

We need to start over. Let’s delete the existing deployment and re-run the pod creation. Go to kubernetes-test folder and run,

kubectl get deployment

# You'll see star-wars-deployment

kubectl delete deployment star-wars-deployment

# deployment.apps "star-wars-deployment" deleted

kubectl get deployment

# You should not see star-wars-deployment

kubectl apply -f sw-deployment.yaml

# deployment.apps/star-wars-deployment created

kubectl get pod

# You should see two pods in "Running" STATUS

Now we have two pods running. Now what?

Service
Now we need to connect to these pods. Each pod has an associated IP address, but if we attempt to use these IPs to connect, it will become unmanageable when the number of Worker machines and prods increases. Here is where the concept of Service comes. It is an abstraction to access Pods without relying on lower-level details like IP addresses.

Note: Read <https://kubernetes.io/docs/concepts/services-networking/service/> to learn more on Service

In the kubernestes-test folder create another file sw-service.yaml with the following content

```
apiVersion: v1
kind: Service
metadata:
  name: sw-service
spec:
  selector:
    app: star-wars
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 31000
```

For the kind Service apiVersion should be v1

metadata.name is the name for the service. It is important to note that spec.selector.app is the exact label we gave for Pods we created above. By the above service, we are giving access to targetPort: 3000 which is the containerPort of each Pod. Also, note that this service has exposed port: 3000 .

By setting type: LoadBalancer we are stating this service as an external service with an external IP address. The nodePort: 31000 is the port that can be used to access this service using the external IP. (note that there is a range for the nodePort between 30000–32767)

Now run the following commands,

kubectl apply -f sw-service.yaml

# service/sw-service created

kubectl get service

# Observe that sw-service created

minikube service sw-service --url

# <http://<external_service_ip>:31000>

Now run curl <http://<external_service_ip>:31000> and see that you can get your favorite Star Wars character. This means we have successfully deployed the NodeJS application in the Kubernetes cluster. Yeyyy!!!

ConfigMap
If you remember, we defined an environment variable MAX_STAR_WARS_CHARACTERS in the Node server. I added this with a purpose. The reason being, introducing the idea of ConfigMap; another Object type in Kubernetes. A ConfigMap can be used to handle ENVs related to a container.

Let’s add another file envs.yaml in the kubernetes-test folder with the content.

apiVersion: v1
kind: ConfigMap
metadata:
  name: sw-envs
data:
  maxStarWarsCharacters: "10"
Just as above we have defined apiVersion, kind and metadata attributed. The data field contains the ENVs we need to inject into the containers.

Now there should be a way to inject this ENV to the starwars-node . It is done by adding container.env field in the sw-deployment.yaml . Add the env section to the .yaml file.

```
...
          ports:
            - containerPort: 3000
          imagePullPolicy: Never
          env:
            - name: MAX_STAR_WARS_CHARACTERS
              valueFrom:
                configMapKeyRef:
                  name: sw-envs
                  key: maxStarWarsCharacters
```

ConfigMap has to create first, therefore run the following commands

```
kubectl apply -f envs.yaml
kubectl apply -f sw-deployment.yaml
```

Now curl <http://<external_service_ip>:31000> and see that you now have 10 Star Wars characters.

And that’s how you can work with minikube and local Docker images.
