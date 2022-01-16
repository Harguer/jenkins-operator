# jenkins-operator V0.7 running in k3s
This Setup is for K3s running in 1 node, this is just for educational purpuse
 - Create namespace jenkins
 - Install the CRD
 - Install the operator pod
 - Install the ClusterRoleBinding for the NFS volumen used by the backup
 - create the PV and the PVC for the backup
 - Create a ConfigMap (I did not have the time to configure ldpap and the authorization stuff, will try to do it later)
 - Install jenkins pod instance with base plugins, seed jobs and the backup settings for the jobs

## Installing the operator, follow the instructions [here](https://jenkinsci.github.io/kubernetes-operator/docs/getting-started/latest/installing-the-operator/) or copy paste these two commands:  
```
kubectl create namespace jenkins
kubens jenkins
kubectl apply -f https://raw.githubusercontent.com/jenkinsci/kubernetes-operator/master/config/crd/bases/jenkins.io_jenkins.yaml 
kubectl apply -f https://raw.githubusercontent.com/jenkinsci/kubernetes-operator/master/deploy/all-in-one-v1alpha2.yaml
kubectl get pods
```

## updated rbac.yaml  [here](https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/master/deploy/rbac.yaml)

## To have a backups feature, enable NFS locally in your server/machine 
```
mkdir /nfs-vol
chmod 777 /nfs-vol
echo '/nfs-vol *(rw,sync,no_subtree_check,no_root_squash)' >> /etc/exports
systemctl enable nfs-server.service
systemctl start nfs-server.service

kubectl apply -f nfs_class.yaml
kubectl apply -f rbac.yaml
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f configmap.yaml
kubectl apply -f jenkins_instance.yaml
```

## Deploying it:
```
[jenkins@local]$ 
[jenkins@local]$ time while read line;do echo doing $line; $line; sleep 1;echo; done < <(egrep "kubectl|kubens" README.md)
doing kubectl create namespace jenkins
namespace/jenkins created

doing kubens jenkins
Context "default" modified.
Active namespace is "jenkins".

doing kubectl apply -f https://raw.githubusercontent.com/jenkinsci/kubernetes-operator/master/config/crd/bases/jenkins.io_jenkins.yaml
customresourcedefinition.apiextensions.k8s.io/jenkins.jenkins.io configured

doing kubectl apply -f https://raw.githubusercontent.com/jenkinsci/kubernetes-operator/master/deploy/all-in-one-v1alpha2.yaml
serviceaccount/jenkins-operator created
role.rbac.authorization.k8s.io/leader-election-role created
rolebinding.rbac.authorization.k8s.io/leader-election-rolebinding created
role.rbac.authorization.k8s.io/jenkins-operator created
rolebinding.rbac.authorization.k8s.io/manager-rolebinding created
deployment.apps/jenkins-operator created

doing kubectl get pods
NAME                                READY   STATUS              RESTARTS   AGE
jenkins-operator-8576d58874-zqqzx   0/1     ContainerCreating   0          1s

doing kubectl apply -f nfs_class.yaml
storageclass.storage.k8s.io/managed-nfs-storage created

doing kubectl apply -f rbac.yaml
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created

doing kubectl apply -f pv.yaml
persistentvolume/jenkins-bck-storage-pv created

doing kubectl apply -f pvc.yaml
persistentvolumeclaim/jenkins-bck-storage-pvc created

doing kubectl apply -f configmap.yaml
configmap/jenkins-operator-user-configuration created

doing kubectl apply -f jenkins_instance.yaml
jenkins.jenkins.io/jenkins created


real	1m58.062s
user	1m39.947s
sys	0m0.704s
[jenkins@local]$ k get pods,secrets,svc,pv,pvc,jenkins
NAME                                         READY   STATUS    RESTARTS   AGE
pod/jenkins-operator-8576d58874-zqqzx        1/1     Running   0          5m52s
pod/jenkins-jenkins                          2/2     Running   0          5m43s
pod/seed-job-agent-jenkins-f696bf684-z5mw2   1/1     Running   0          76s

NAME                                          TYPE                                  DATA   AGE
secret/default-token-9wlrz                    kubernetes.io/service-account-token   3      7m41s
secret/jenkins-operator-token-prb4l           kubernetes.io/service-account-token   3      5m53s
secret/jenkins-operator-jenkins-token-2dgsp   kubernetes.io/service-account-token   3      5m44s
secret/jenkins-operator-credentials-jenkins   Opaque                                4      5m44s

NAME                                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
service/jenkins-operator-http-jenkins    ClusterIP   10.43.91.178   <none>        8080/TCP    5m43s
service/jenkins-operator-slave-jenkins   ClusterIP   10.43.15.166   <none>        50000/TCP   5m43s

NAME                                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                             STORAGECLASS          REASON   AGE
persistentvolume/jenkins-bck-storage-pv   10Gi       RWO            Retain           Bound    jenkins/jenkins-bck-storage-pvc   managed-nfs-storage            5m48s

NAME                                            STATUS   VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS          AGE
persistentvolumeclaim/jenkins-bck-storage-pvc   Bound    jenkins-bck-storage-pv   10Gi       RWO            managed-nfs-storage   5m47s

NAME                         AGE
jenkins.jenkins.io/jenkins   5m44s
[jenkins@local]$ 
```

## Extract jenkins-operator password from the secret
```
kubectl get secrets jenkins-operator-credentials-jenkins -o jsonpath={.data.password}| base64 -d ;echo
```

## Open a web browser and copy the ip from `kubectl get svc` paste it in browser and use port 8080, use jenkins-operator and the password above   
![alt text](https://github.com/Harguer/jenkins-operator/blob/main/jenkins-screenshot.png?raw=true)


