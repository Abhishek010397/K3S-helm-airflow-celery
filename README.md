# K3S-helm-airflow-celery

-------------------------------------------------------------------------------------------------------------------------------------------
# Installing K3s
   K3s supports 3 node cluster
   
   
        1. master( use t2.medium EC2 instance 4GB RAM 2 CORE)
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

# Install helm3 on K3s cluster

 Commands executed on master node
 
     curl -L https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
----------------------------------------------------------------------------------------------------------------------------------------     
 To confirm version of helm
    
    $ helm version
     
----------------------------------------------------------------------------------------------------------------------------------------

#  Now run following commands to give permissions to tiller for performing operations

   
     $ kubectl -n kube-system create serviceaccount tiller
     $ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
     
 ---------------------------------------------------------------------------------------------------------------------------------------



# Add Helm Chart repository

    $ helm repo add stable https://kubernetes-charts.storage.googleapis.com/
    $ helm search repo stable
----------------------------------------------------------------------------------------------------------------------------------------

Install Applications on Helm Chart

    $ kubectl config get-contexts
    
 for switching to desired context
    
    $ kubectl config use-context k3s
    
    $ helm repo update
----------------------------------------------------------------------------------------------------------------------------------------

# NGINX Ingress controller can be easily installed from official Helm chart stable/nginx-ingress repository.
 
    
    $ helm show chart stable/nginx-ingress
    $ helm install nginx-ingress stable/nginx-ingress 
    
 if this throws an ERROR:Kubernetes cluster unreachable
   configure:-
       
       export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
 and then install
       
       $ helm install nginx-ingress stable/nginx-ingress 
 to configure installation
    
       $ helm ls
 ---------------------------------------------------------------------------------------------------------------------------------------
 
 # Airflow on Kubernetes
 
 
 We will make following changes to make our airflow installation useful in a n enterprise setting:

Create a namespace
Change the source of DAG files in the helm chart
Set up Active Directory authentication for airflow (Optional)

----------------------------------------------------------------------------------------------------------------------------------------

# Create a namespace

    kubectl config set-context dev1 --namespace=airflow
 if the above command throws an error as: 
 
    error: open /etc/rancher/k3s/k3s.yaml.lock: permission denied
    
 then execute kubectl command with sudo privilleges.
    
----------------------------------------------------------------------------------------------------------------------------------------

# clone the charts repository:

    git clone https://github.com/helm/charts.git
    cd charts/stable/airflow
    
 Open values.yaml in a text editor and modify following sections:
 
    dags:
    ##
    ## mount path for persistent volume.
    ## Note that this location is referred to in airflow.cfg, so if you change it, you must update airflow.cfg accordingly.
    path: /usr/local/airflow/dags
    ##
    ## Set to True to prevent pickling DAGs from scheduler to workers
    doNotPickle: false
    ##
    ## Configure Git repository to fetch DAGs
    git:
    ##
    ## url to clone the git repository
    url: <your_own_git_url>
    ##
    ## branch name, tag or sha1 to reset to
    ref: master
    
---------------------------------------------------------------------------------------------------------------------------------------
  
 #  Create airflow-image
        
        mkdir docker-airflow
        cd docker-airflow
        vi Dockerfile
        
   create a docker file for an airflow image and then build it and push to your repository
   
         docker build . -t <dockerfile_name>
         docker tag <img_id> hubusername:image_name
         docker push hubusername/image_name
         
   then go back to the values.yaml file and under image section:
         
         image:
         ##
         ## docker-airflow image
         repository: <your docker hub repository containing the airflow image>
         ##
         ## image tag
         tag: latest
         ##
         ## Image pull policy
         ## values: Always or IfNotPresent
         pullPolicy: IfNotPresent
         ##
         ## image pull secret for private images
         pullSecret:
----------------------------------------------------------------------------------------------------------------------------------------
       
  #   Installation
     
        helm install stable/airflow -f values.yaml --generate-name
   
   if this throws an an error:kubernetes cluster not found/unreachable,then configure export command in the same directory as:
      
       export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
       
   and then procced with installation
   
   -------------------------------------------------------------------------------------------------------------------------------------
 # Access the service
       
       kubectl get service
   to view the port number:
   
       kubectl describe service<airflow-web-service>
       
  # Now use the port number to access it from browser
  
       kubectl port-forward service/airflow-web 8080:8080
  
