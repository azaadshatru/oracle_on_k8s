# Oracle Express Edition on Open Source Kubernetes Cluster (k8s)
Installing Oracle Express 18.4.0.0 on K8s cluster

Install nfs-client storage class
=================================

Set nfs server
---------------
nfs host = lab1.example.com
nfs path = /nfs/oracle

nfs server
-----------
# vi /etc/exports
# /nfs/oracle     *(rw,sync,no_root_squash,insecure)
# systemctl restart nfs-server.service

nfs client - on all Kube Nodes
-------------------------------
# mkdir -p /nfs/oracle
# vi /etc/fstab
lab1.example.com:/nfs/oracle   /nfs/oracle       nfs defaults 0 0
showmount -e lab1.example.com
# mount -a

**Install Storage Class using helm charts**

# helm repo add stable https://charts.helm.sh/stable
# helm repo update
# helm install my-nfs-client stable/nfs-client-provisioner --namespace nfs-client --set nfs.server=lab1.example.com --set nfs.path=/nfs/oracle
# kubectl get storageclass
NAME         PROVISIONER                                       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   cluster.local/nfs-client-provisioner-1626944060   Delete          Immediate           true                   2m13s

Make Storage Class Default
--------------------------
# kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
# kubectl get sc
NAME                   PROVISIONER                                          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   cluster.local/my-nfs-client-nfs-client-provisioner   Delete          Immediate           true                   5d7h

In case you want to remove nfs-client storage class
----------------------------------------------------
# helm uninstall my-nfs-client

Installing Oracle On Kubernetes
================================

# mkdir -p ~/oracle
# cd ~/oracle
# git clone 
# kubectl create namespace oracle-namespace --save-config
# kubectl config set-context --current --namespace=oracle-namespace
# sudo docker login container-registry.oracle.com
# kubectl -n oracle-namespace create secret generic regcred \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson

# kubectl get secret -n oracle-namespace
secret/regcred created

# kubectl get secret -n oracle-namespace
NAME                  TYPE                                  DATA   AGE
default-token-qxvkn   kubernetes.io/service-account-token   3      20h
regcred               kubernetes.io/dockerconfigjson        1      10s

# kubectl create configmap oradb --from-env-file=oracle.properties -n oracle-namespace
configmap/oradb created

# kubectl apply -f 18xe_deployment_nfs-client.yaml -n oracle-namespace
deployment.apps/oracle18xe created
persistentvolumeclaim/ora-data184-claim created
persistentvolumeclaim/ora-setup184-claim created
persistentvolumeclaim/ora-startup184-claim created
service/oracle18xe created

# kubectl get pvc -n oracle-namespace
NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ora-data184-claim      Bound    pvc-0decaa1f-8d37-43c5-b906-7988e1567bb9   10Gi       RWO            nfs-client     11m
ora-setup184-claim     Bound    pvc-7447bd71-22fd-466b-96d4-7695619e56fd   1Gi        RWO            nfs-client     11m
ora-startup184-claim   Bound    pvc-fcbe3d8c-8d98-4bf6-8443-61ef473aaa3e   1Gi        RWO            nfs-client     11m

# kubectl get svc -n oracle-namespace
NAME         TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
oracle18xe   NodePort   10.105.239.154   <none>        1521:19419/TCP,5500:30944/TCP   11m

By using the command above, we can see that our Oracle listener (1521) is available externally on Port 19419 and Enterprise Manager (5500) is available on 30944.

# kubectl get pods -n oracle-namespace
NAME                         READY   STATUS    RESTARTS   AGE
oracle18xe-c89b5d998-7lcmg   1/1     Running   0          5m44s

# kubectl -n oracle-namespace logs -f oracle18xe-c89b5d998-7lcmg
ORACLE PASSWORD FOR SYS AND SYSTEM: OracDB#2168
Specify a password to be used for database accounts. Oracle recommends that the password entered should be at least 8 characters in length, contain at least 1 uppercase character, 1 lower case character and 1 digit [0-9]. Note that the same password will be used for SYS, SYSTEM and PDBADMIN accounts:
Confirm the password:
Configuring Oracle Listener.
Listener configuration succeeded.
Configuring Oracle Database XE.
Enter SYS user password:
*********
Enter SYSTEM user password:
*********
Enter PDBADMIN User Password:
*********
Prepare for db operation
7% complete
Copying database files
29% complete
Creating and starting Oracle instance
30% complete
31% complete
34% complete
38% complete
41% complete
43% complete
Completing Database Creation
47% complete
50% complete
Creating Pluggable Databases
54% complete
71% complete
Executing Post Configuration Actions
93% complete
Running Custom Scripts
100% complete
Database creation complete. For details check the logfiles at:
 /opt/oracle/cfgtoollogs/dbca/XE.
Database Information:
Global Database Name:XE
System Identifier(SID):XE
Look at the log file "/opt/oracle/cfgtoollogs/dbca/XE/XE.log" for further details.

Connect to Oracle Database using one of the connect strings:
     Pluggable database: oracle18xe-c89b5d998-7lcmg/XEPDB1
     Multitenant container database: oracle18xe-c89b5d998-7lcmg
Use https://localhost:5500/em to access Oracle Enterprise Manager for Oracle Database XE
The Oracle base remains unchanged with value /opt/oracle
#########################
DATABASE IS READY TO USE!
#########################
The following output is now a tail of the alert.log:
Pluggable database XEPDB1 opened read write
Completed: alter pluggable database XEPDB1 open
2021-08-17T07:55:10.221620+00:00
XEPDB1(3):CREATE SMALLFILE TABLESPACE "USERS" LOGGING  DATAFILE  '/opt/oracle/oradata/XE/XEPDB1/users01.dbf' SIZE 5M REUSE AUTOEXTEND ON NEXT  1280K MAXSIZE UNLIMITED  EXTENT MANAGEMENT LOCAL  SEGMENT SPACE MANAGEMENT  AUTO
XEPDB1(3):Completed: CREATE SMALLFILE TABLESPACE "USERS" LOGGING  DATAFILE  '/opt/oracle/oradata/XE/XEPDB1/users01.dbf' SIZE 5M REUSE AUTOEXTEND ON NEXT  1280K MAXSIZE UNLIMITED  EXTENT MANAGEMENT LOCAL  SEGMENT SPACE MANAGEMENT  AUTO
XEPDB1(3):ALTER DATABASE DEFAULT TABLESPACE "USERS"
XEPDB1(3):Completed: ALTER DATABASE DEFAULT TABLESPACE "USERS"
2021-08-17T07:55:13.197092+00:00
ALTER PLUGGABLE DATABASE XEPDB1 SAVE STATE
Completed: ALTER PLUGGABLE DATABASE XEPDB1 SAVE STATE
2021-08-17T08:04:36.147161+00:00
XEPDB1(3):Resize operation completed for file# 10, old size 358400K, new size 3                                                                                                          78880K

# select instance_name, host_name, to_char(startup_time,'dd/mm/yy hh24:mi:ss') as Startup, status from v$instance;
