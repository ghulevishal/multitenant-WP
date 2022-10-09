# Multi-Tenanncy with Kubernetes 
The deployment is intended to demonstrate of how we can deploy these microservices in Kubernetes with seperate namespaces that directly map to each tenant on-boarded, then in-turn the frontend application which exposes the website where users can browse items, add them to the cart, and purchase them, and accessible independently through different subdomain for each tenant. 

     http://<tenant-name>.<saas-domain> (e.g. http://test.demo.xyz)

-------
To be able to get the deployment up and running, you need to have Kubernetes cluster up and running with Ingress controller. 
If you already have minikube up. 

Once you already have the cluster up and running, let's deploy the first with helm. Here we are using simple nginx application.

    $ helm install test ./helmchart

If everything configured properly, then you will have tenant `test` be able to access their isolated deployment through their registered subdomain 

    http://test.demo.xyz

We are acheiving this with the ingress object.

----------
[Helm]
-------
Helm is a tool to help you manage Kubernetes application, There are several ways to install it as described in the [Quickstart Guide](https://helm.sh/docs/using_helm/#installing-helm). We will go with one of them as follows.

    $ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get > get_helm.sh
    $ chmod 700 get_helm.sh
    $ ./get_helm.sh
    
Once you finish above steps, you can start deploying your Kubernetes application through helm, happy helming!

### Namespace

Coming back to the deployment command we run earlier. Lets do this with dry run this time, so that there will be no resource be allocated, this is super helpful if we are validating whether our scripts will work as expected and see the result of the instruction inside the Kubernetes cluster.
      
    $ helm install --dry-run --debug test ./helmchart
  
There are lots of thing going on once you run the command, particularly you can observe the namespace. What it is, is basically a way for you as the cluster administrator allocate an environment for your certain group of users, in our case those groups are our tenant, further readings [here](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). 

    apiVersion: v1
    kind: Namespace
    metadata:
      name: {{ .Release.Name }}
      labels: 
        tenant: {{ .Release.Name }}

The preceding snippets are taken from helmchart/templates/namespace.yaml. Here we are using `{{ .Release.Name }}` as the variable to name the namespace, this variable is filled with the value you passed after the `helm install` command, which in this case it is `test`.

And if you open other files, you will find that those deployment and service and ingress are to be deployed in the namespace you created by declaring this line `namespace: {{ .Release.Name }}`

### Values
Values is being passed to the templates that we are going to deploy our Kubernetes application from values.yaml. For example, as you see where we are assigning container port `containerPort: {{ .Values.adsService.ports.containerPort }}`, you can find the value of the containerport variable if you examine the values.yaml file, so does other parameters defined for adservice use as follows.

    frontend:
      terminationGracePeriodSeconds: 5
      image:
        repository: nginx
        tag: 1.14.2
        pullPolicy: IfNotPresent
      ports:
        containerPort: 80
      healthChecks:
        readinessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/"
            port: 80
        livenessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/"
            port: 80
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
        limits:
          cpu: 200m
          memory: 128Mi
      service:
        type: ClusterIP
        name: http
        port: 80

This also applies to other deployment and services too. We do it this way so that the configuration is easier to maintain, and reproducible in the future.

### Setting Up New Tenant

Earlier we have setup tenant "test" in our cluster, with a very easy command. This is important, because in order to maintain massive amount of client that we always wanted for our business, we need automation to play part in our deployment.

Let deploy new tenant environment and call it testtenant.

    $ helm install testtenant ./helmchart

### Rolling Out New Update Independently

One of the point of having single tenancy design as mention earlier, is to have the ability to scale independently and isolated, so that if our business decided to tap in new market that is not originally planned (e.g. new region with different policies and regulations or supporting customisation) that our application is not designed to accommodate, we are free to adjust without impacting the other tenant.

Having said that, maintaining different deployment that has different configurations and applications can greatly add complexity to our infrastructure. Again Kubernetes to the rescue, with the following command, we can start deploying the new customisation.

    $ helm upgrade testtenant ./helmchart --set=frontend.image.repository=nginx --set=frontend.image.tag=alpine

After executing the command above we will update the base image of the frontend application that is being used by tenant "testtenant" to `nginx:alpine` without impacting tenant "test" that still use the default image.  

Now we know, that we can deploy separately for each of our tenants, these tenants could use different custom image that is coming from different development line and could be maintained by different development team. The diagram below shows how we could alter the pipeline if we decided to deploy those custom images.

