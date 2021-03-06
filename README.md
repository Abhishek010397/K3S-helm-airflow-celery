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
     
Execute below commans only after k3s-agent is setup on workers

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
# Add Helm Chart repository

    $ helm repo add stable https://kubernetes-charts.storage.googleapis.com/  (stable chart)
    $ helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/  (incubator chart)
    $ helm search repo stable
----------------------------------------------------------------------------------------------------------------------------------------

# Install Applications on Helm Chart

    $ kubectl config get-contexts
    
# for switching to desired context
    
    $ kubectl config use-context k3s
    
    $ helm repo update
----------------------------------------------------------------------------------------------------------------------------------------

# NGINX Ingress controller can be easily installed from official Helm chart stable/nginx-ingress repository.
 
    
    $ helm show chart stable/nginx-ingress
    $ helm install nginx-ingress stable/nginx-ingress 
    
 if this throws an ERROR:Kubernetes cluster unreachable
   *configure:-
       
       export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
 *and then install
       
       $ helm install nginx-ingress stable/nginx-ingress 
 *to configure installation
    
       $ helm ls
 ---------------------------------------------------------------------------------------------------------------------------------------
 
 # Airflow on Kubernetes
 
 
 We will make following changes to make our airflow installation useful in a n enterprise setting:

Create a namespace
Change the source of DAG files in the helm chart


----------------------------------------------------------------------------------------------------------------------------------------

# Create a namespace

    kubectl config set-context dev1 --namespace=airflow
 if the above command throws an error as: 
 
    error: open /etc/rancher/k3s/k3s.yaml.lock: permission denied
    
 then execute kubectl command with sudo privilleges.
    
----------------------------------------------------------------------------------------------------------------------------------------

# Edit the values.yaml file :

Open values.yaml in a text editor and modify following sections: *there are certain changes made in this yaml file,go through the note present at the top of that file*
 
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
    
 refer:-  
          
          git clone https://github.com/helm/charts.git
          cd charts/stable/airflow
                  
---------------------------------------------------------------------------------------------------------------------------------------
  
 #  Create airflow-image
        
 Docker should be installed in the cluster:-https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04
 
        mkdir docker-airflow
        cd docker-airflow
        vi Dockerfile
        
   create a docker file for an airflow image and then build it and push to your repository
   
         docker build . -t <dockerfile_name>
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
 # Provide RBAC permissions
   *In Kubernetes, granting roles to a user or an application-specific service account is a best practice to ensure that your        application is operating in the scope that you have specified.
   
   Edit the value.yaml file under the section rbac:
   
       rbac:
       ##
       ## Specifies whether RBAC resources should be created
      
         create: true
         rules:
           cluster:
            - apiGroups:
                - ""
              resources:
                - configmaps
              verbs:
               - get
               - list
               - watch
               - patch
               - update
               - create
               - delete
 
 For more info :- https://helm.sh/docs/topics/rbac/
 
 ---------------------------------------------------------------------------------------------------------------------------------------
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
       
 ---------------------------------------------------------------------------------------------------------------------------------------
 
 # Now for setting up DASK
 
          helm repo add dask https://helm.dask.org/    # add the Dask Helm chart repository
          helm repo update                             # get latest Helm charts
          helm install dask/dask                     # deploy standard Dask chart
 
   *Verify Deployment*
        
        
             helm list
             kubectl get pods
             kubectl get services
             
       ou can use the addresses under EXTERNAL-IP to connect to your now-running Jupyter and Dask systems.

Notice the name bald-eel. This is the name that Helm has given to your particular deployment of Dask. We could, for example, have multiple Dask-and-Jupyter clusters running at once, and each would be given a different name.


  *Connect to Dask and Jupyter*/
  
When we ran kubectl get services, we saw some externally visible IPs:
eg:-

     mrocklin@pangeo-181919:~$ kubectl get services
     NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                       AGE
     bald-eel-jupyter     LoadBalancer   10.11.247.201   35.226.183.149   80:30173/TCP                  2m
     bald-eel-scheduler   LoadBalancer   10.11.245.241   35.202.201.129   8786:31166/TCP,80:31626/TCP   2m
     kubernetes           ClusterIP      10.11.240.1     <none>           443/TCP                       48m

        
  
  
We can navigate to these services from any web browser. Here, one is the Dask diagnostic dashboard, and the other is the Jupyter server. You can log into the Jupyter notebook server with the password, dask.

We can create a notebook and create a Dask client from there. The DASK_SCHEDULER_ADDRESS environment variable has been populated with the address of the Dask scheduler. This is available in Python in the config dictionary.
 
   For more Info
   Refer:- https://docs.dask.org/en/latest/
  
