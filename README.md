# Using Traefik to redirect traffic with K3S

Traefik is included as part of the services of K3S, so you dont need any extra installation

### Note: download the yaml files instead of typing, look under code.

## Deploying a simple website
To make this deployment you will need to be familiar with YAML. YAML configuration files are used, and that is what we will use in this article. We will start at the top and create our configuration files in a top-down approach.

## Deployment configuration
The configuration is shown below, and the explanation follows. I typically use the samples from the Kubernetes documentation as a starting point and then modify them to suit my needs. For example, the configuration below was modified after copying the sample from the deployment docs.

Create a file, mysite.yaml, with the following contents:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mysite-nginx
      labels:
        app: mysite-nginx
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: mysite-nginx
      template:
        metadata:
          labels:
            app: mysite-nginx
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80

The important parts, we have named our deployment mysite-nginx with an app label of mysite-nginx as well. We have specified that we want one replica which means there will only be one pod created. We also specified one container, which we named nginx. We specified the image to be nginx. This means, on deployment, k3s will download the nginx image from DockerHub and create a pod from it. Finally, we specified a containerPort of 80, which just means that inside the container the pod will listen on port 80.

I emphasized "inside the container" above because it is an important distinction. As we have the container configured, it is only accessible inside the container, and it is further restricted to an internal network. This is necessary to allow multiple containers to listen on the same container ports. In other words, with this configuration, some other pod could listen on its container port 80 as well and not conflict with this one. To provide formal access to this pod, we need a service configuration.

## Service configuration
In Kubernetes, a service is an abstraction. It provides a means to access a pod or set of pods. One connects to the service and the service routes to a single pod or load balances to multiple pods if multiple pod replicas are defined.

The service can be specified in the same configuration file, and that is what we will do here. Separate configuration areas with ---. Add the following to mysite.yaml:

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: mysite-nginx-service
    spec:
      selector:
       app: mysite-nginx
      ports:
        - protocol: TCP
          port: 80

In this configuration, we have named our service mysite-nginx-service. We provided a selector of app: mysite-nginx. This is how the service chooses the application containers it routes to. Remember, we provided an app label for our container as mysite-nginx. This is what the service will use to find our container. Finally, we specified that the service protocol is TCP and the service listens on port 80.

## Ingress configuration
The ingress configuration specifies how to get traffic from outside our cluster to services inside our cluster. Remember, k3s comes pre-configured with Traefik as an ingress controller. Therefore, we will write our ingress configuration specific to Traefik. Add the following to mysite.yaml ( and don’t forget to separate with ---):

    ---
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: mysite-nginx-ingress
      annotations:
        kubernetes.io/ingress.class: "traefik"
    spec:
      rules:
      - http:
          paths:
          - path: /
            backend:
              serviceName: mysite-nginx-service
              servicePort: 80

In this configuration, we have named the ingress record mysite-nginx-ingress. And we told Kubernetes that we expect traefik to be our ingress controller with the kubernetes.io/ingress.class annotation.

In the rules section, we are basically saying, when http traffic comes in, and the path matches / (or anything below that), route it to the backend service specified by the serviceName mysite-nginx-service, and route it to servicePort 80. This connects incoming HTTP traffic to the service we defined earlier.

Something to deploy
That is really it as far as configuration goes. If we deployed now, we would get the default nginx page, but that is not what we want. Let’s create something simple but custom to deploy. Create the file index.html with the following contents:

    <html>
    <head><title>K3S!</title>
      <style>
        html {
          font-size: 62.5%;
        }
        body {
          font-family: sans-serif;
          background-color: midnightblue;
          color: white;
          display: flex;
          flex-direction: column;
          justify-content: center;
          height: 100vh;
        }
        div {
          text-align: center;
          font-size: 8rem;
          text-shadow: 3px 3px 4px dimgrey;
       }
      </style>
    </head>
    <body>
      <div>Hello from K3S!</div>
    </body>
    </html>

We have not yet covered storage mechanisms in Kubernetes, so we are going to cheat a bit and just store this file in a Kubernetes config map. This is not the recommended way to deploy a website, but it will work for our purposes. Run the following:

    kubectl create configmap mysite-html --from-file index.html

This command creates a configmap resource named mysite-html from the local file index.html. This essentially stores a file (or set of files) inside a Kubernetes resource that we can call out in configuration. It is typically used to store configuration files (hence the name), so we are abusing it a bit here. In a later article, we will discuss proper storage solutions in Kubernetes.

With the config map created, let’s mount it inside our nginx container. We do this in two steps. First, we need to specify a volume, calling out the config map. Then we need to mount the volume into the nginx container. Complete the first step by adding the following under the spec label, just after containers in mysite.yaml:

      volumes:
      - name: html-volume
        configMap:
          name: mysite-html

This tells Kubernetes that we want to define a volume, with the name html-volume and that volume should contain the contents of the configMap named html-volume (which we created in the previous step).

Next, in the nginx container specification, just under ports, add the following:

        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html

This tells Kubernetes, for the nginx container, we want to mount a volume named html-volume at the path (in the container) /usr/share/nginx/html. Why /usr/share/nginx/html? That is where the nginx image serves HTML from. By mounting our volume at that path, we have replaced the default contents with our volume contents.

For reference, the deployment section of the configuration file should now look like this:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mysite-nginx
      labels:
        app: mysite-nginx
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: mysite-nginx
      template:
        metadata:
          labels:
            app: mysite-nginx
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80
            volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html
          volumes:
          - name: html-volume
            configMap:
              name: mysite-html

Deploy it!
Now we are ready to deploy! We can do that with:

    kubectl apply -f mysite.yaml

You should see something similar to the following:

    deployment.apps/mysite-nginx created
    service/mysite-nginx-service created
    ingress.networking.k8s.io/mysite-nginx-ingress created

This means that Kubernetes created resources for each of the three configurations we specified. Check on the status of the pods with:

    kubectl get pods

If you see a status of ContainerCreating, give it some time and run kubectl get pods again. Typically, the first time, it will take a while because k3s has to download the nginx image to create the pod. After a while, you should get a status of Running.

Try it!
Once the pod is running, it is time to try it. Open up a browser and type kmaster into the address bar.


## Deploying a second website

Another one
So now we have a whole k3s cluster running a single website. But we can do more! What if we have another website we want to serve on the same cluster? Let’s see how to do that.

Again, we need something to deploy. It just so happens that my dog has a message she has wanted the world to know for some time. So, I crafted some HTML just for her (available from the samples zip file). Again, we will use the config map trick to host our HTML. This time we are going to poke a whole directory (the html directory) into a config map, but the invocation is the same.

    kubectl create configmap mysecond-html --from-file html

Now we need to create a configuration file for this site. It is almost exactly the same as the one for mysite.yaml, so start by copying mysite.yaml to mydog.yaml. Now edit mydog.yaml to be:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mysecond-nginx
      labels:
        app: mysecond-nginx
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: mysecond-nginx
      template:
        metadata:
          labels:
            app: mysecond-nginx
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80
            volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html
          volumes:
          - name: html-volume
            configMap:
              name: mysecond-html  
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: mysecond-nginx-service
    spec:
      selector:
        app: mysecond-nginx
      ports:
        - protocol: TCP
          port: 80
    ---
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: mysecond-nginx-ingress
      annotations:
        kubernetes.io/ingress.class: "traefik"
        traefik.frontend.rule.type: PathPrefixStrip
    spec:
      rules:
      - http:
          paths:
          - path: /mysecond
            backend:
              serviceName: mysecond-nginx-service
              servicePort: 80

We can do most of the edits by simply doing a search and replace of mysite to mydog. The two other edits are in the ingress section. We changed path to /mysecond and we added an annotation, traefik.frontend.rule.type: PathPrefixStrip.

The specification of the path /mysecond instructs Traefik to route any incoming request that requests a path starting with /mysecond to the mydog-nginx-service. Any other path will continue to be routed to mysite-nginx-service.

The new annotation, PathPrefixStrip, tells Traefik to strip off the prefix /mysecond before sending the request to mysecond-nginx-service. We did this because the mydog-nginx application doesn’t expect a prefix. This means we could change where the service was mounted simply by changing the prefix in the ingress record.

Now we can deploy like we did before:

    kubectl apply -f myseocond.yaml
    
And now, my second’s message should be available at http://kmaster/mysecond/
