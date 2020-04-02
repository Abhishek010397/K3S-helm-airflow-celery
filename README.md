# K3S-helm-airflow-celery

-------------------------------------------------------------------------------------------------------------------------------------------
# Installing K3s
   K3s supports 3 node cluster
    1. master
    2. node
    3. node
    
-------------------------------------------------------------------------------------------------------------------------------------------

# On Master

The first line specifies in which mode we would like to write the k3s configuration (required when not running commands as root) and the second line actually says k3s not to deploy its default load balancer named servicelb and proxy traefik, instead we will install         manually   metalb as load balancer and nginx as proxy which are in my opinion better and more widely used.
   
     $ export K3S_KUBECONFIG_MODE="644"
     $ export INSTALL_K3S_EXEC=" --no-deploy servicelb --no-deploy traefik"
   
The next command simply downloads and executes the k3s installer. The installation will take into account the environment variables    set just before.  
  
    $ curl -sfL https://get.k3s.io | sh -

      
Verify the status  

    $ sudo systemctl status k3s
     
To get the details of the nodes       
         
    $ kubectl get nodes -o wide
  
To get the details of all the services deployed         
         
    $ kubectl get pods -A -o wide
  
Each agent will require an access token to connect to the server, the token can be retrieved with the following commands        
  
    $ sudo cat /var/lib/rancher/k3s/server/node-token
  
         
         
----------------------------------------------------------------------------------------------------------------------------------------


# On Node(Workers)

The first line specifies in which mode we would like to write the k3s configuration (required when not running command as root)         and the second line provide the k3s server endpoint the agent needs to connect to. Finally, the third line is an access token to         the k3s server saved previously.

     $ export K3S_KUBECONFIG_MODE="644"
     $ export K3S_URL="https://<master_ip>:6443"
     $ export K3S_TOKEN="<master_generated_token"

Run the installer      
      
     $ curl -sfL https://get.k3s.io | sh -
 
      
Verify the status        
 
     $ sudo systemctl status k3s-agent
 
     
      
----------------------------------------------------------------------------------------------------------------------------------------
