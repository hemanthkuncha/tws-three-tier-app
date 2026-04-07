faced issue while frontend accessing the API from my browser (which is in diffrenet network)

resolved it by exposing the api deployment and used the exposed port

kubernetes/frontend.yaml -- >> (http://<NODE_IP>:PORT/api/tasks)

---
untill now used the hard coded ip values in (env) in frontend.yaml

when the deploying hardware ip changes then i have to rebuild the image to access the backend through browser due to the ENV is directly adding while build process

and we can added a manual DNS entry in local

now we can access frontend - http://10.0.0.40:30101 
                             http://10.0.0.40:30102/api/tasks
***
edit /etc/hosts -- >> 10.0.0.40 myapp.local (in the vm that browser able to route to that ip) -->> http://myapp.local:30102/api/tasks

now we can access frontend - http://myapp.local:30101 
                             http://myapp.local:30102/api/tasks
___
for solving the Hard-Port-Numbering: We need Ingress & LoadBalancer

adding the ingress controller to cluster and using the ingress routing paths to access

adding the metalLB baremetal to cluster and using ipPool to have Virtual LoadBalancer

now we can access frontend - http://myapp.local
                             http://myapp.local:30102/api/tasks-----------------
                             
For SSL-TLS : HTTPS 
1. Manual
2. CertManager

now we can access frontend - https://myapp.local
                             https://myapp.local:30102/api/tasks
