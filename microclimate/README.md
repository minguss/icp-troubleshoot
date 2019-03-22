# Microclimate 설치 
### (1.6기준)
먼저 Namespace를 생성한다.
편의상 devops와 microclimate-pipeline-deployments라는 네임스페이스를 생성했다.

devops는 microclimate ide가 설치 될 namespace이고, microclimate-pipeline-deployments는 배포 될 namespace가 되겠다.
``` bash
kubectl create namespace devops
kubectl config set-context $(kubectl config current-context) --namespace=devops
kubectl create namespace microclimate-pipeline-deployments
```
![image1](../images/microclimate1.png)  
배포 될 namespace인 devops namespace에 secret키를 생성한다.
secret키가 없으면 docker pull을 해와서 기동할 때 정상적으로 Conatainer 생성/동작을 하지 않는다.

``` bash
kubectl create secret docker-registry microclimate-registry-secret --docker-server=mycluster.icp:8500 --docker-username=admin --docker-password=admin --docker-email=null -n devops
```
![image2](../images/microclimate2.png)  
clusterimagepolicy를 일부 수정해야한다. 아래 명령어로 접근하여 내용을 추가한다.
``` bash
kubectl edit clusterimagepolicy ibmcloud-default-cluster-image-policy
 
  - name: docker.io/maven:*
  - name: docker.io/lachlanevenson/k8s-helm:*
  - name: docker.io/jenkins/*
```
![image3](../images/microclimate3.png)  
microclimate-pileline-deployments의 serviceaccount에 pull secret patch를 한다.
``` bash
kubectl describe serviceaccount default --namespace microclimate-pipeline-deployments
kubectl create secret docker-registry microclimate-pipeline-secret --docker-server=mycluster.icp:8500 --docker-username=admin --docker-password=admin --docker-email=null --namespace=microclimate-pipeline-deployments
kubectl patch serviceaccount default --namespace microclimate-pipeline-deployments -p '{"imagePullSecrets": [{"name": "microclimate-pipeline-secret"}]}'
kubectl create secret docker-registry microclimate-pipeline-secret --docker-server=mycluster.icp:8500 --docker-username=admin --docker-password=admin --docker-email=null --namespace=devops
kubectl patch serviceaccount default --namespace devops -p '{"imagePullSecrets": [{"name": "microclimate-pipeline-secret"}]}'
kubectl create secret docker-registry microclimate-pipeline-secret --docker-server=mycluster.icp:8500 --docker-username=admin --docker-password=admin --docker-email=null --namespace=default
kubectl patch serviceaccount default --namespace default -p '{"imagePullSecrets": [{"name": "microclimate-pipeline-secret"}]}'
```
![image4](../images/microclimate4.png)  
serviceaccount patch를 완료하면 helm secret도 등록해야한다. 먼저 cloudctl로 icp cluster에 로그인한다.
``` bash
cloudctl login -a https://mycluster.icp:8443 --skip-ssl-validation
```
![image5](../images/microclimate5.png)  
cloudctl로 로그인한 뒤 helm secret을 생성한다.
``` bash
kubectl create secret generic microclimate-helm-secret --from-file=cert.pem=/root/.helm/cert.pem --from-file=ca.pem=/root/.helm/ca.pem --from-file=key.pem=/root/.helm/key.pem
```
![image6](../images/microclimate6.png)  
그리고 설치하기 전에 PersistentVolume을 생성해야한다.
Microclimate은 ReadWriteMany를, Jenkins는 ReadWriteOnce로 각각 PV를 생성해야한다.
![image7](../images/microclimate7.png)  
그리고 helm repository를 추가해야하는데 아래와 같은 command로 repo를 추가하고 update를 한다.
``` bash
### Helm Repository 추가
helm repo add ibm-charts https://raw.githubusercontent.com/IBM/charts/master/repo/stable/
 
### Helm Repo update
helm repo update
```

이후에 helm cli로 microclimate을 설치한다.
``` bash
helm install --name microclimate --namespace devops --set global.rbac.serviceAccountName=micro-sa,jenkins.rbac.serviceAccountName=pipeline-sa,hostName=microclimate.<proxy-ip>.nip.io,jenkins.Master.HostName=jenkins.<proxy-ip>.nip.io,persistence.useDynamicProvisioning=false,jenkins.Pipeline.Registry.Url=mycluster.icp:8500 ibm-charts/ibm-microclimate --tls

###ㅅㅣㄴㄱㅠ신규
helm install --name microclimate --namespace devops --set global.rbac.serviceAccountName=micro-sa,jenkins.rbac.serviceAccountName=pipeline-sa,global.ingressDomain=192.168.1.141.nip.io,jenkins.Pipeline.Registry.URL=mycluster.icp:8500/devops ibm-charts/ibm-microclimate --tls
```
![image8](../images/microclimate8.png)  

----
helm을 설치해야 할 경우
``` bash
helm delete microclimate --purge --tls
```

``` bash
helm upgrade microclimate -f overrides.yaml ibm-charts-public/ibm-microclimate --reuse-values --tls
```
