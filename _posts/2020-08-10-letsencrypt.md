---
layout: post
title:  "Why you should not create stand alone certs with Let’s Encrypt"
author: sandro
categories: [ DEV ]
image: assets/images/lets-encrypt.png
---
Yeah you heared right. You should not do that. And the reason is, because your address bar will sooner or later look like that:

![NOTSECURE]({{ site.baseurl }}/assets/images/afterworkbeer-not-secure.jpg)

And who wants an unsafe afterwork beer, right? You guessed it: Nobody.

#### In the beginning there was a shortli(o)ved cert

Let’s wrap it up a bit. Let’s Encrypt is probably the savior of the 21 century. Well, maybe that is a bit over-exaggerated but in the end they probably gave us DEVs the most convenient and obviously the cheapest solution to secure our web applications.
So, having a http:// url without the famous “s”, is just something we don’t want to see anymore. Even all browsers automatically try to redirect your http:// urls to https://. Which make sense in most of the cases, if it is at least a productive app. So, let’s create a cert!

First and foremost, we install something called certbot (Let’s Encrypt’s ACME client).

For an Ubuntu 20.04 it would look like this:
```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository universe
sudo apt-get update
sudo apt-get install certbot
```

And then we simply create our standalone cert like so:
```
sudo certbot certonly --manual -d mysite.yourdomain.ch --agree-tos --no-bootstrap --manual-public-ip-logging-ok --preferred-challenges http-01
```

We have to put some info in the prompts and then we get our wanted cert files in the location: `/etc/letsencrypt/live/mysite.yourdomain.ch/`. So, we take those files and probably use it in one of our apps. In my case I have a kubernetes cluster so I would do something like that:
```
kubectl create secret tls mysite-yourdomain-ch-tls --cert=fullchain1.pem --key=privkey1.pem
```
And now we gonna use that secret in one of our ingress, like so:
```
 apiVersion: networking.k8s.io/v1beta1
 kind: Ingress
 metadata:
  name: mysite-yourdomain-ingress
 spec:
  tls:
    - hosts:
      - mysite.yourdomain.ch
      secretName: mysite-yourdomain-ch-tls
  rules:
  - host: mysite.yourdomain.ch
    http:
      paths:
      - path: /
        backend:
          serviceName: mysite
          servicePort: 80
```
And then what next? Nothing next. We have now a secure endpoint on https://mysite.yourdomain.ch. So, our job is done and we can continue doing more interesting stuff.

And then, time passes…

And after 90 days we get what? Yeah you know it, don’t you?
 
And who do we blame? Us. Because we didn’t do what we once read in an article. And because we didn’t care.
After all the thing we used to create the cert was called ‘certbot‘. So, why not use that ‘bot’ to make the work for us.
In kubernetes we can install something that is called an issuer or clusterIssuer. And as the name implies, this guy is issuing certs for us.
Let’s create a Let’s Encrypt clusterIssuer that creates and renew certs for us in the background:
```
# install cert-manager with helm
kubectl create namespace cert-manager 
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.1 \
  --set installCRDs=true

# create an issuer
```
create a file ‘clusterIssuer.yaml’ with following content:
```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
name: letsencrypt-prod
spec:
acme:
# The ACME server URL
server: https://acme-v02.api.letsencrypt.org/directory
# Email address used for ACME registration
email: yourmail@gmail.com
# Name of a secret used to store the ACME account private key
privateKeySecretRef:
name: letsencrypt-prod
# Enable the HTTP-01 challenge provider
solvers:
- http01:
ingress:
class: nginx
```

And then create the issuer:
```
kubectl apply -f clusterIssuer.yaml
```
Now you can just add the following two lines to every ingress and the cert-manager will automatically create the certs for you in the background.
```
...
metadata:
  ...
  annotations:
   kubernetes.io/ingress.class: nginx
   cert-manager.io/cluster-issuer: letsencrypt-prod
...
```
#### Finally
We have certbot doing the work for us. And the best thing about all this is; cert-manager even renews the certs for you!

![SECURE]({{ site.baseurl }}/assets/images/afterworkbeer-secure.jpg)

And now let's have a beer!

. . . Until next time
