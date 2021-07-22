# Running microservices in Amazon EKS with AWS App Mesh and Kong
by Mikhail Shapirov | on 03 DEC 2020 | in Amazon Elastic Kubernetes Service, AWS App Mesh, Containers | Permalink |  Share
This post was created in collaboration with Claudio Acquaviva, Solution Engineer, Kong, and Morgan Davies, Kong Alliances.

A service mesh is transparent infrastructure layer that has become a common architectural pattern for intra-service communication. By combining Amazon EKS and AWS App Mesh, you form a powerful platform for your microservices, addressing technical requirements that occur in service-to-service communication, including load balancing, service discovery, observability, access control, tracing, health checks, and circuit breakers.

A modern enterprise solution requires clear management controls for the following categories:

API Management covering external traffic ingress to the API endpoints.
Service management capabilities focusing on operational controls and service health.
While service meshes primarily address the second category, the ingress is no less important and can benefit from a solution that supports cluster-wide policies such as throttling, application and user authentication, request logging and tracing, and data caching. In addition to these polices, the ingress is the layer that enables you to monetize your APIs by capturing usage, attaching billing systems, and generating alerts that go beyond operational concerns.

While it is possible to achieve this by stitching tools outside of the cluster perimeter, the Kong for Kubernetes Ingress Controller provides a solution that will protect your service mesh running side by side with your application services, leveraging Kubernetes capabilities like HPA, self-healing, RBAC, and cert-manager, among others.

This post will explore how to use Amazon EKS, AWS App Mesh, and Kong for Kubernetes to implement and protect a service mesh. The problem space that we will address is not just about managing your APIs and external traffic; we will also cover deeper integration scenarios. Not only will we handle the ingress in a Kubernetes-native way, we will also make it part of the service mesh itself, improving your observability, security, and traffic control.

Enter AWS App Mesh and Kong for Kubernetes Ingress Controller
AWS App Mesh is a fully managed service that customers can use to implement a service mesh. This service makes it easy to manage internal service-to-service communication across multiple types of compute infrastructure. Kong for Kubernetes is responsible for controlling the traffic going through the ingresses that expose the service mesh to external consumers by defining, applying, and enforcing policies to the ingresses.

Kong for Kubernetes supports the following capabilities:

Scalability: Based on the Kong API gateway, it’s responsible for managing the ingresses. It is common for applications to experience significant fluctuations in volume of traffic, affecting your ingress as well. Kong for Kubernetes is taking advantage of standard Kubernetes scalability controls like Horizontal Pod Autoscaler (HPA) and will scale seamlessly with the demand.
Security: Leverages Kubernetes namespace-based RBAC model to ensure consistent access controls. These controls are essential to segregate responsibilities between platform, API, and application teams, which handle their part in the software delivery and operations. For example, application teams restricted to their individual namespaces must still be able to define ingress objects, while access to the ingress controller and API management components can be restricted to the dedicated team(s).
Extensibility: An extensive plugin ecosystem offers a variety of options to protect your service mesh, such as OpenID Connect and mutual TLS authentication and authorization, rate-limiting, IP restrictions, and self-service credential registration through the Kong Enterprise Developer Portal.
Observability: It can be fully integrated with monitoring, tracing and logging tools like Prometheus, Jaeger, and AWS CloudWatch.
Here’s the Kong for Kubernetes architecture diagram:



Kong for Kubernetes Architecture
The following diagram describes the Kong for Kubernetes architecture:

Kong architecture diagram

The Kong for Kubernetes pod contains two containers:

The Kong Gateway container represents the data plane responsible for processing API traffic and enforcement of policies defined by the ready-to-use plugins available in Kong for Kubernetes.
The controller container represents the control plane that translates Kubernetes manifests and CRDs into Kong configuration, removing the need for separate administration of proxy and Kubernetes configuration.
In front of Kong for Kubernetes, there is a Classic Load Balancer (CLB) or Network Load Balancer (NLB) exposing the Kong Gateway to the external consumers. Furthermore, Kong for Kubernetes is protecting all services behind it, including ClusterIP services running inside the Kubernetes cluster or external services exposed in your cluster.

Prerequisites
Before starting this process, ensure the following prerequisites are ready:

An EKS 1.15 or higher cluster is already deployed. For this exercise, eksctl was used
Kubectl 1.15 or higher installed locally
Helm V3
Curl or any other HTTP client
Solution deployment
The deployment is an evolution of the DJ Service Mesh Application, adding the ingress controller layer on top of it. Kong for Kubernetes provides an extensive list of plugins to implement numerous policies, such as authentication, log processing, caching, and more.

To get started, let’s implement an API key-based security layer and rate-limiting policies to control the ingress consumption.

Step 1: Deploy your DJ service mesh application
Follow the following steps described in the EKS workshop to deploy the DJ service mesh application:

Deploy DJ App
Install App Mesh integration
Port DJ App to App Mesh
These steps will install a simple solution consisting of metal and jazz microservices (two versions of each), install AWS App Mesh components, including CRDs and App Mesh controller, which among other functions, acts as an admission controller, injecting envoy proxy sidecar to the deployed pods. The prod namespace will be configured to enable automatic injection of sidecar containers, providing the required level of abstraction to control traffic policies.

The meshification of the application will result in abstraction of communication between the DJ pod and the existing microservices. Instead of the going directly to the endpoints of jazz and metal microservices, the API flow will go through the new Jazz and Metal virtual services, which in turn use the new virtual router objects to control the traffic. At the end of the process, the logical architecture will look like this:



Before we start, let’s check what virtual services, virtual routers and virtual nodes are available in the cluster:

$ kubectl get virtualservices -n prod
NAME    ARN                                                                                              AGE
jazz    arn:aws:appmesh:us-west-2:<AWS_ACCOUNT>:mesh/dj-app/virtualService/jazz.prod.svc.cluster.local    2m39s
metal   arn:aws:appmesh:us-west-2:<AWS_ACCOUNT>:mesh/dj-app/virtualService/metal.prod.svc.cluster.local   2m38s

$ kubectl get virtualrouters -n prod
NAME           ARN                                                                                  AGE
jazz-router    arn:aws:appmesh:us-west-2:<AWS_ACCOUNT>:mesh/dj-app/virtualRouter/jazz-router_prod    2m54s
metal-router   arn:aws:appmesh:us-west-2:<AWS_ACCOUNT>:mesh/dj-app/virtualRouter/metal-router_prod   2m53s

$ kubectl get virtualnodes -n prod
NAME       ARN                                                                            AGE
dj         arn:aws:appmesh:us-west-2:<AWS_ACCOUNT>:mesh/dj-app/virtualNode/dj_prod         3m8s
jazz-v1    arn:aws:appmesh:us-west-2:<AWS_ACCOUNT>:mesh/dj-app/virtualNode/jazz-v1_prod    3m4s
jazz-v2    arn:aws:appmesh:us-west-2:<AWS_ACCOUNT>:mesh/dj-app/virtualNode/jazz-v2_prod    2m41s
metal-v1   arn:aws:appmesh:us-west-2:<AWS_ACCOUNT>:mesh/dj-app/virtualNode/metal-v1_prod   3m4s
metal-v2   arn:aws:appmesh:us-west-2:<AWS_ACCOUNT>:mesh/dj-app/virtualNode/metal-v2_prod   2m40s
Note, that the DJ node is redundant since it does not route any traffic and mainly serves testing purposes. What is important is that we have used the App Mesh to implement a canary release of jazz-v2 and metal-v2 services, routing 5% of all traffic to version 2 of both services:

$ kubectl describe virtualrouter jazz-router -n prod 
...
  Routes:
    Http Route:
      Action:
        Weighted Targets:
          Virtual Node Ref:
            Name:  jazz-v1
          Weight:  95
          Virtual Node Ref:
            Name:  jazz-v2
          Weight:  5
...
Why is it important? Regardless of how your services are consumed, through the external ingress or internally by other services, all communication will honor the canary release sending 5% of all traffic to the new version.

That is all great, but how do you test the canary release as part of your development process? It is clear that you can run curl commands from the DJ pod, but that will not work for your CI/CD pipeline and proper API testing strategies that demand that the consumer of the API must be external (similar to the real world).

This highlights the fact, that the solution in its current state should be improved if you are planning to expose your APIs to the outside world. Proper API management demands that we add controls over API clients (identified by the API key), rate limiting and, eventually, usage capturing, and billing.

Step 2: Deploy Kong for Kubernetes Ingress Controller
In this blog post, we will replace the redundant DJ node that can be fully replaced with Kong for Kubernetes. We will define an ingress object that will expose all of our services to external consumers.

The following diagram shows the final topology:



Kong for Kubernetes namespace

The Kubernetes namespace where Kong for Kubernetes components will reside has to be configured with proper labels in order to make it part of the existing mesh and to apply automatic injection of the App Mesh sidecar:

kubectl create namespace kong
kubectl label namespace kong mesh=dj-app appmesh.k8s.aws/sidecarInjectorWebhook=enabled
Kong for Kubernetes virtual node

Virtual node declaration for Kong for Kubernetes:

apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: kong
  namespace: kong
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/instance: kong
  listeners:
    - portMapping:
        port: 80
        protocol: http
  backends:
    - virtualService:
        virtualServiceRef:
          name: jazz
          namespace: prod
    - virtualService:
        virtualServiceRef:
          name: metal
          namespace: prod
  serviceDiscovery:
    dns:
      hostname: kong-kong-proxy.kong.svc.cluster.local
Notice that the declaration is:

Selecting the Kong for Kubernetes pod to be installed in the step.
Defining Jazz and Metal services as the backend, since they are the only allowed ingress points.
Setting the Kong for Kubernetes’ Service FQDN as the DNS Service Discovery.
Use kubectl to submit it:

kubectl apply -f https://raw.githubusercontent.com/Kong/aws-blogposts/master/K4K8S+AWSAppMesh/kongvirtualnode.yml
Check the virtual nodes again. The Kong for Kubernetes specific one means it’s been incorporated by the “dj-app” Mesh.

$ kubectl get virtualnodes --all-namespaces
NAMESPACE NAME  ARN  AGE
kong  kong  arn:aws:appmesh:us-west-2::mesh/dj-app/virtualNode/kong_kong  62s
prod  dj  arn:aws:appmesh:us-west-2::mesh/dj-app/virtualNode/dj_prod  22m
prod  jazz-v1 arn:aws:appmesh:us-west-2::mesh/dj-app/virtualNode/jazz-v1_prod 22m
prod  jazz-v2 arn:aws:appmesh:us-west-2::mesh/dj-app/virtualNode/jazz-v2_prod 21m
prod  metal-v1 arn:aws:appmesh:us-west-2::mesh/dj-app/virtualNode/metal-v1_prod 22m
prod  metal-v2 arn:aws:appmesh:us-west-2::mesh/dj-app/virtualNode/metal-v2_prod 21m
Kong for Kubernetes installation

We will use Helm to add the kong chart repository and install Kong for Kubernetes:

helm repo add kong https://charts.konghq.com 
helm repo update 
helm install -n kong kong kong/kong --version "1.11.0" --set ingressController.installCRDs=false
By default App Mesh will not allow egress from the nodes except to the nodes that are explicitly defined in the mesh. This will prevent the ingress controller container to communicate with the API server, so the deployment of the Kong pod will be failing. To address this aspect, we will use an App Mesh feature, which allows to bypass egress filtering for containers running under security context with (by default) UID 1337. Setting this security context option for the ingress-controller enables it to communicate with the API server, while the rest of the nodes in the service mesh will be restricted to communicate only with the nodes that are defined in the mesh:

kubectl patch deploy -n kong kong-kong -p '{"spec":{"template":{"spec":{"containers":[{"name":"ingress-controller","securityContext":{"runAsUser": 1337}}]}}}}'
Once the patch is applied, let’s validate the deployment and make sure that all containers are running:

$ kubectl get pods -n kong
NAME READY STATUS RESTARTS AGE
kong-kong-5b4499bc4-rgxnd 3/3 Running 0 13m
Since the provisioned service that exposes the ingress is of “type: LoadBalancer”, we get an ELB instance along with it:

$ kubectl get service -n kong
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kong-kong-proxy LoadBalancer 10.100.18.222 adf320d20effa44d4b49ca2cf279e0b8-240857585.{region}.elb.amazonaws.com 80:32445/TCP,443:32024/TCP 5d19h
Verify, the Envoy sidecar has been injected in the Kong for Kubernetes pod by listing all containers and images in the pod:

$ kubectl get po -n kong -o jsonpath='{range .items[*]}{"pod: "}{.metadata.name}{"\n"}{range .spec.containers[*]}{"\tname: "}{.name}{"\n\timage: "}{.image}{"\n"}{end}'
pod: kong-kong-6f784b6686-qlrvp
name: ingress-controller
image: kong-docker-kubernetes-ingress-controller.bintray.io/kong-ingress-controller:1.0
name: proxy
image: kong:2.1
name: envoy
image: 840364872350.dkr.ecr.us-west-2.amazonaws.com/aws-appmesh-envoy:v1.15.1.0-prod
Step 3: Define Ingress to expose and protect the service mesh
Service configuration

Kong for Kubernetes configures targets based on the endpoints of the corresponding Kubernetes service. That means that Kong itself communicates to pods directly and is using service abstraction as a discovery mechanism for the endpoints. This allows Kong to perform load-balancing bypassing the extra hop of going through the kube proxy and applying optimized load balancing algorithms.

For AWS App Mesh, it is important to note that service communication is only allowed through fully qualified domain names of services. For example, routing to jazz.prod.svc.cluster.local is permitted. However, directly invoking the service by its host name (e.g. curl jazz:9080) will not succeed. Ability to address services by host names is on the AWS App Mesh roadmap, so in the future this setup will be a lot simpler.

Luckily, Kong allows to supply configuration that will address both constraints. The following command will do both, set up the service for upstream for Kong ingress, and instruct Kong to use configuration that enables FQDN name instead of service host name for routing.

kubectl apply -f https://raw.githubusercontent.com/Kong/aws-blogposts/master/K4K8S%2BAWSAppMesh/fqdn-service-routing.yaml
Kong for Kubernetes Ingress

The next steps is to define the Kubernetes ingress object with the routing rules that will expose both jazz and metal virtual services to external traffic with proper security and traffic controls provided by Kong. Ingress object is a standard Kubernetes object:

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: djingress
  namespace: prod
  annotations:
    konghq.com/strip-path: "true"
    kubernetes.io/ingress.class: kong
    konghq.com/override: do-not-preserve-host
spec:
  rules:
  - http:
      paths:
        - path: /dj/jazz
          backend:
            serviceName: jazz
            servicePort: 9080
        - path: /dj/metal
          backend:
            serviceName: metal
            servicePort: 9080
kubectl create -f https://raw.githubusercontent.com/Kong/aws-blogposts/master/K4K8S+AWSAppMesh/dj_ingress.yml
The annotation konghq.com/strip-path: "true" removes the extended path like “/dj/jazz” from the request before sending it to target virtual service.
The annotation konghq.com/override: do-no-preserve-host points to the configuration object that removes the original host for the request. Combined with the FQDN annotation applied to the service, it allows the sidecar to route the request based on the right authority.
The configuration option “do-not-preserve-host” referenced in the ingress definition refers to the following:

apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: do-not-preserve-host
  namespace: prod
route:
  preserve_host: false
Apply this configuration option to the ingress in the prod namespace:

kubectl create -f https://raw.githubusercontent.com/Kong/aws-blogposts/master/K4K8S+AWSAppMesh/kongingress-dontpreservehost.yml
Check the ingress with kubectl also. Notice that this ingress is using the same load balancer as the one provisioned for Kong proxy.

$ kubectl get ingress -n prod
NAME     HOSTS   ADDRESS  PORTS AGE
djingress *      {name.region}.elb.amazonaws.com 
Consume the ingress using the external address specified for the ingress:

curl {your ingress address}/dj/jazz
["Astrud Gilberto","Miles Davis"]
Run a loop to see the canary release in action:

while [ 1 ];
  do curl http://{your ingress address}/dj/jazz/  
  echo
done

["Astrud Gilberto","Miles Davis"]
["Astrud Gilberto","Miles Davis"]
["Astrud Gilberto","Miles Davis"]
["Astrud Gilberto","Miles Davis"]
["Astrud Gilberto (Bahia, Brazil)","Miles Davis (Alton, Illinois)"]
["Astrud Gilberto","Miles Davis"]
["Astrud Gilberto","Miles Davis"]
You can change the URL path to /dj/metal and do the same for the metal service.

With this step we have exposed our services externally preserving the canary functionality and in addition we got an important layer for ingress and API management on top of our service mesh. The ingress fully implements the DJ functionality routing traffic to the underlying virtual services, allowing us to drop the DJ virtual node.

Step 4: Apply rate-limiting policy
With the ingress in place, it’s necessary to define policies to control its consumption. The first one is rate-limiting. The process to apply policies to an ingress is very simple:

Declare and create a policy
Patch the ingress with an annotation
The rate-limiting policy shown below will allow three requests per minute:

apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rl-by-minute
  namespace: prod
config:
  minute: 3
  policy: local
plugin: rate-limiting
Apply this policy:

kubectl create -f https://raw.githubusercontent.com/Kong/aws-blogposts/master/K4K8S+AWSAppMesh/ratelimiting.yml
Once the policy created it can be applied to the ingress that needs it:

kubectl patch ingress djingress -n prod -p '{"metadata":{"annotations":{"konghq.com/plugins":"rl-by-minute"}}}'
If you try to consume the service more than three times a minute, you will receive an error. Note, that with the Kong Ingress you can control traffic to any DJ service that will be part of the underlying implementation (assuming later we add services like “classical” or “country”).

while [ 1 ] 
  do curl {your ingress address}/dj/jazz/
  echo 
done
["Astrud Gilberto","Miles Davis"]
["Astrud Gilberto","Miles Davis"]
["Astrud Gilberto","Miles Davis"]
{
"message":"API rate limit exceeded"
}
You can refer to to this page to check the plugins provided by Kong to integrate with an extensive list of log processing, real-time monitoring, and tracing tools.

Step 5: Define the API key security policy
Similarly to the rate-limiting policy, we will need to create the policy first and then apply this policy to the ingress:

apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: apikey
  namespace: prod
plugin: key-auth
kubectl create -f https://raw.githubusercontent.com/Kong/aws-blogposts/master/K4K8S+AWSAppMesh/apikey.yml
Apply the API key policy by adding another annotation to the ingress. Notice that this time we’re applying both policies to it:

kubectl patch ingress djingress -n prod -p '{"metadata":{"annotations":{"konghq.com/plugins":"apikey, rl-by-minute"}}}'
You will now see an error when trying to consume the ingress:

curl {your ingress address}/dj/jazz
{
"message":"No API key found in request"
}
Provision a key and associate it to a consumer:

kubectl create secret generic consumerapikey -n prod --from-literal=kongCredType=key-auth --from-literal=key=kong-secret
Let’s create the consumer and associate it with the consumerapikey:

apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: consumer1
  namespace: prod
  annotations:
    kubernetes.io/ingress.class: kong
username: consumer1
credentials:
- consumerapikey
kubectl apply -f https://raw.githubusercontent.com/Kong/aws-blogposts/master/K4K8S+AWSAppMesh/consumer.yml
Now that the consumer can be identified, let’s consume the ingress with the API key (note the header passed to the curl command):

curl {your ingress address}/dj/jazz -H 'apikey:kong-secret'
["Astrud Gilberto","Miles Davis"]
Conclusion
Kong for Kubernetes and AWS App Mesh make it easy to run services by providing consistent visibility and network traffic controls for services built across multiple platforms. You can learn more about products showcased in this blog through the official documentation: AWS App Mesh and Kong for Kubernetes.

In the next posts, we will show even deeper integration with the AWS ecosystem by adding observability, tracing, and monitoring to the picture, covering integration with AWS X-Ray and Elasticsearch. Additionally, we will focus on authentication and authorization aspects with OIDC and Amazon Cognito.

Feel free to change the policies used in this post and experiment further implementing policies like caching, log processing, OIDC-based authentication, canary, GraphQL integration, and more with the extensive list of plugins provided by Kong.

All the declarations used in this post are available on GitHub.
