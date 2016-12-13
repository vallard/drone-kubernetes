# Run Drone.io on Kubernetes

To get CI/CD in your environment a good way is to run [drone.io](http://drone.io) on kubernetes.  

```
Updated for Drone 0.5
Drone 0.4 available in 0.4 subdirectory 
```

## Prerequisites
* Working Kubernetes
* NGINX controller or Ingress controller

## Installation instructions

### Create new Github Application

Register a new app with Github [here](https://github.com/settings/applications/new)

This will give you a client ID and client Secret.  Make note of these. 

* The Authorization callback URL should be ```http://drone.yourdomain.com/authorize``` where ```yourdomain``` is your own thing. 

#### Create the service

```
kubectl create -f drone-svc.yaml
```
Notice that in this service we are using ```NodePort: 30991```  since we are doing this on Metacloud.  This ```NodePort``` is used by nginx to attach to with reverse proxy.  

#### Create and deploy secrets

Open the ```drone-secrets.yaml``` file and put in your own credentials you got when you registered your Github application from the first step. 

You have to base64 encode all these values: 

```
echo -n "/var/lib/drone/drone.sqlite" | base64
```

See the examples in the ```drone-secrets.yaml``` file comments. 

```
kubectl create -f drone-secrets.yaml
```

#### Create the drone deployment

This file uses the secrets file.  

```
kubectl create -f drone.yaml
```

If all goes well drone should be up.  

## Nginx Reverse Proxy
Using OpenStack I have a seperate node running nginx.  This node has the configuration that looks as follows for the drone section: 

```nginx
http {
  upstream drone {
    server 10.106.2.24:30991;
    server 10.106.2.32:30991;
    server 10.106.2.29:30991;
    server 10.106.2.33:30991;
    server 10.106.2.26:30991;
    server 10.106.2.35:30991;            
  }
  
  server {
    listen 80;
    server_name drone.example.com www.drone.example.com;
    location / {
	   proxy_set_header X-Real-IP $remote_addr;
    	proxy_set_header X-Forwarded-For $remote_addr;
    	proxy_set_header X-Forwarded-Proto $scheme;
    	proxy_set_header Host $http_host;
    	proxy_set_header Origin "";	
     	proxy_pass http://drone;
		proxy_redirect off;
	   	proxy_http_version 1.1;
    	proxy_buffering off;
		chunked_transfer_encoding off;
    }
  }
}
```
## Troubleshooting

* Make sure your containers have outbound internet access.  E.g: spin up busybox and run ```nslookup github.com``` if this doesn't work you'll get infinite loop.  I solved this by running ```iptables -t nat -A POSTROUTING ! -d 10.0.0.0/8 -o ens3 -j MASQUERADE```


