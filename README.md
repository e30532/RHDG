This document describes a simple use case of Red Hat Data Grid with liberty.

1. deploy OCP4.13.4 (3 master, 3 worker)    
2. login to the ocp console and deploy Data Grid operator     
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/63ce715b-5d54-455a-a57f-691d97ce855c">
   
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/671b01c3-e033-4968-88b7-8f1842cf0b93">    
   
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/380fa611-8807-4160-aa32-b57850009012">    
   


4. create a NFS storage class   
    3.1. On NFS server, create a NFS directory and expose it to the 6 OCP nodes.
    ```
    # mkdir -p /data/mynfs1
    # vi /etc/exports
    # exportfs -ra
    # exportfs -v
    /data         	10.0.0.0/8(sync,no_wdelay,hide,no_subtree_check,sec=sys,rw,insecure,no_root_squash,no_all_squash)
    ```
    3.2. Download [storage-class.yml](https://raw.githubusercontent.com/e30532/RHDG/main/storage-class.yml), [persitent-volumne-claim.yml](https://raw.githubusercontent.com/e30532/RHDG/main/persitent-volumne-claim.yml), [rbac.yml](https://raw.githubusercontent.com/e30532/RHDG/main/rbac.yml) and [deployment.yml](https://raw.githubusercontent.com/e30532/RHDG/main/deployment.yml)   
    3.3. Update two points in deployment.xml with the NFS server IP address
    3.4. create the resources which are related to NFS dynamic storage provisioning
    ```
    # oc apply -f storage-class.yml -f persitent-volumne-claim.yml -f rbac.yml -f deployment.yml
    ```
5. create a infinispan cluster
    4.1. expose to route 
6. 

