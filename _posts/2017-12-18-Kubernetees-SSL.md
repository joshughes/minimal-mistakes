---
layout: single
title:  "Kubernetes SSL with Letencrypt"
date:   2017-12-18 21:56:21 -0500
categories: Kubernetes
---


## Lets Encrypt with Kube Lego! 

Traditional SSL cert managment is expensive and time consuming, so whenever possible I reach for Lets Encrypt to give me free certs! The certs given are only good for 90 days which makes you solve the renewal process automation up front. So your not scrambling when that three year cert expires. [Kube Lego](https://github.com/jetstack/kube-lego) is an excellent tool that does pretty much all the heavy lifting for your. I will explain below the basics of getting it deployed to your Kubernetes cluster, and some things I have learned using the tool. 

## Deploy with Helm 

[Helm](https://github.com/kubernetes/helm) is meant to be a package manager for your Kubernetes cluster. It deserves its own post, but in short I think it does a good job but may not be 100% ready for prime time in some cases. To deploy Kube Lego though, it works like a champ and can help you rapidly get the tool deployed to all your clusters! 

Kube Lego works with the both Nginx and Google Container Engine Ingress controllers. It will get the certificate from Lets Encrypt and ensure it is installed correctly. 

### Setting up helm on your cluster

Follow the instructions on instructions for installing the helm client on your system found [HERE](https://github.com/kubernetes/helm/blob/master/docs/install.md). 

Once installed you just need to init the server side of helm into the Kubernetes Cluster. 

```shell
helm init 
```

Now you have an active Helm install! 

### Install Kube-Lego with helm 

First you need to create a file called `values.yaml` with the content below. 

```yaml
## kube-lego configuration
## Ref: https://github.com/jetstack/kube-lego
##
config:
  ## Email address to use for registration with Let's Encrypt
  ##
  LEGO_EMAIL: YOUR_EMAIL_HERE@YOU.com 

  ## Let's Encrypt API endpoint
  ## Production: https://acme-v01.api.letsencrypt.org/directory
  ## Staging: https://acme-staging.api.letsencrypt.org/directory
  ##
  LEGO_URL: https://acme-v01.api.letsencrypt.org/directory

  ## kube-lego port
  ##
  LEGO_PORT: 8080

## kube-lego image
image:
  repository: jetstack/kube-lego
  tag: 0.1.5
  pullPolicy: IfNotPresent

## Node labels for pod assignment
## Ref: https://kubernetes.io/docs/user-guide/node-selection/
##
nodeSelector: {}

## Annotations to be added to pods
##
podAnnotations: {}

replicaCount: 1

## kube-lego resource limits & requests
## Ref: https://kubernetes.io/docs/user-guide/compute-resources/
##
resources: {}
```

The `values.yaml` file allows us to make the helm install more easily repeatable and also ensure some settings are correct. `LEGO_EMAIL` and `LEGO_URL` are the two important ones. By default the Helm Chart will point you to the staging Lets Encrypt environment instead of production, giving you invalid certs. `LEGO_EMAIL` is needed so Lets Encrypt can send you emails if something goes wrong and your SSL certificates for a domain are going to expire. 

You can always go to the chart and see if there are any more configuration values or a new image tag that has been tested with the cart. To do that you just need to go to the [stable chart github repository](https://github.com/kubernetes/charts/blob/master/stable/kube-lego/values.yaml). 

Now we are ready to use Helm to deploy the Kube Lego service! 

In the same directory where you created `values.yaml` run the following. 
*Note*
`--namespace inf` is optional but I like to use namespaces to seperate my services as logically as possible. Infrastructure based services I always put into a namespace called `inf`.

```shell
helm install --namespace inf -f values.yml stable/kube-lego
```

Helm should give you some output and Kube Lego is now running in your cluster! 


## Using Kube Lego in your Ingress Resources 

Annotations drive most of the customer configuration in Kubernetes and its no different in this case. Lets take the following basic Ingress as an example of a Ingress Resource we want to add SSL to. 

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  namespace: devopsdeepstate
spec:
  rules:
  - host: microservice1.devopsdeepstate.com
    http:
      paths:
      - backend:
          serviceName: microservice1
          servicePort: 80
        path: /
```

To turn on Kube Lego we need the annotation `kubernetes.io/tls-acme: "true"` which lets Kube Lego know to do the following. 

1. Requests a SSL Certificate from Lets Encrypt 
2. Sets up an [ACME HTTP Challenge](https://letsencrypt.org/how-it-works/) response path to verify we own microservice1.devopsdeepstate.com.
3. Responds to Lets Encrypt to ensure we pass the ACME Challenge and Lets Encrypt can issue us a SSL Certificate 
4. Store the private key to that SSL Certificate in a Kubernetes Secret so our Ingress controllers can use it. 

So for step 4 to work we must let Kube Lego know where to store the SSL Certificate. That is done with the following configuration block. 

```yaml
tls:
- hosts:
  - microservice1.devopsdeepstate.com
  secretName: microservice1-devopsdeepstate-com
```

Specifically `secreteName` is the configuration we are talking about now. I like to replace `.` with `-` to make all my SSL Certificate secrets easy to identify when and if I need to look at them for any reason. This block of configuration is actually the same block we would use if we had created the SSL cert manually. The `hosts` portion of this configuration could container one or more DNS names... the certificates will be stored in the same secretName. I would avoid doing this just to keep things well labeled and easier to understand. 

So all together its actually only a small amount of work to get SSL as you can see with the following final example. 

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
  namespace: devopsdeepstate
spec:
  rules:
  - host: microservice1.devopsdeepstate.com
    http:
      paths:
      - backend:
          serviceName: microservice1
          servicePort: 80
        path: /
  tls:
  - hosts:
    - microservice1.devopsdeepstate.com
    secretName: microservice1-devopsdeepstate-com
```

### Gotcha's 

So Kube Lego is a great tool... the only issue I have ever run into is the following scenario. 

Building on our example above, lets say I have an subpath that needs basic authentication. I need a separate Ingress Resource to handle that. It would look like the following. 

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
   ingress.kubernetes.io/auth-secret: basic-auth
   ingress.kubernetes.io/auth-type: basic
    kubernetes.io/ingress.class: nginx
    
  namespace: devopsdeepstate
spec:
  rules:
  - host: microservice1.devopsdeepstate.com
    http:
      paths:
      - backend:
          serviceName: secure_service
          servicePort: 80
        path: /admin
  tls:
  - hosts:
    - microservice1.devopsdeepstate.com
    secretName: microservice1-devopsdeepstate-com
```

I am going to write another blog dedicated to authentication, but for now just notice what is not in the above configuration `kubernetes.io/tls-acme: "true"`. Since both of the ingress share the same host name of `microservice1.devopsdeepstate.com` if we tell Kube Lego to try to go out and request certificates for both it is going to throw an error and crash. You will notice that on their readme they have the following. 

`The secretName statements have to be unique per namespace`

So if we have `kubernetes.io/tls-acme: "true"` turned on and the same secretName we break Kube Lego. The nice thing is that as long as the first Ingress has `kubernetes.io/tls-acme: "true"` we don't have to worry and we can reuse the `microservice1-devopsdeepstate-com` in as many other Ingress Resources as we want. 


## Why SSL All the things?

[Wifi security is broken!](https://gizmodo.com/dont-panic-but-wi-fis-main-security-protocol-has-been-1819501001) You never know how secure the network that your users are on. With SSL it protects your users and shows that you care. It is the first thing I always do when I setup an environment, thanks to great tools like Lets Encrypt and Kube Lego, I can SET IT AND FORGET IT, and move on to more interesting and challenging problems. 




