## Beginner Test

## Prerequisites
- Development instance (Ubuntu 16.04 | Ubuntu 18.04)
- GCP Account
- Install Google Cloud SDK and then setup gcloud command-line on your development box
  - For Debian/Ubuntu : https://cloud.google.com/sdk/docs/quickstart-debian-ubuntu
  - For macOS : https://cloud.google.com/sdk/docs/quickstart-macos
  - For RedHat and CentOS : https://cloud.google.com/sdk/docs/quickstart-redhat-centos
- ubuntu user with passwordless sudo access
- internet access from development box

## Steps
- Clone this project and cd into repo
```
git clone https://github.com/taherbohari/k8s-test.git
cd k8s-test
```

- Install kubectl on your development instance
```
sudo ./beginner/install-prereq
```

- Export all required parameters
```
export PROJECT_ID=<provide your own project_id>
export K8S_CLUSTER_NAME="test-cluster"
export K8S_REGION="asia-south1"
export NODES_PER_ZONE="2"
export MACHINE_TYPE="n1-standard-1"
export K8S_VER="1.12.8-gke.10" <This version should be supported by GKE>
export INGRESS_NAME=<provide your own ingress name>
```
**NOTE : ** Update values as per your requirement

- Create your first gcloud project
```
gcloud projects create ${PROJECT_ID}
```

- Create kubernetes cluster
```
./beginner/setup-gcloud-k8s-cluster
```
**NOTE :** If you get below error. You might need to enable Kubernetes Engine API
```
ERROR: (gcloud.beta.container.clusters.create) ResponseError: code=403, message=Kubernetes Engine API is not enabled for this project. Please ensure it is enabled in Google Cloud Console and try again: visit https://console.cloud.google.com/apis/api/container.googleapis.com/overview?project=k8s-retest to do so.
```

- Setup helm
```
./beginner/setup-helm
```

- Deploy ingress controller
```
./beginner/deploy-ingress-controller
```

- Make note of ExternalIP of LoadBalancer svc
```
kubectl --namespace default get services -o wide ${INGRESS_NAME}-nginx-ingress-controller
```

- Deploy guestbook application
```
./beginner/setup-guestbook-application
```

- Update EXTERNAL_IP in below entries with external ip of ingress LoadBalancer and add them to your instance host file
```
<EXTERNAL_IP> staging-guestbook.mstakx.io
<EXTERNAL_IP> guestbook.mstakx.io
```
**RESULT:** Access guestbook application on above two uri's

- Deploy Horizontal Pod Autoscaler
```
./beginner/deploy-hpa
kubectl get hpa -n staging
```

- Now will see how this horizontal pod autoscaler works

- Check number of guestbook frontend pods running
```
kubectl get pods -l app=guestbook -l tier=frontend -n staging
```
- Scale down pod count to 1
```
kubectl scale deployment frontend --replicas=1 -n staging
```
- Check again number of guestbook frontend pods running
```
kubectl get pods -l app=guestbook -l tier=frontend -n staging
```
- Now run busybox container to generate load on your guestbook app
```
kubectl run -it --restart=Never --image=busybox:1.28 load-generator -n staging -- /bin/sh
```
*NOTE :* If you don't see a command prompt, try pressing enter.

- You are now inside a pod. Run below command to generate a load
```
while true; do wget -q -O- frontend.staging.svc.cluster.local > /dev/null; done
```
- Open an another parallel terminal and run below command to check autoscaling in action
```
kubectl get hpa frontend -n staging -w
```
- Now you can see some load generated on application. You might see kinda same output as below
```
ubuntu@ubuntu:~$ kubectl get hpa frontend -w
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
frontend   Deployment/frontend   1%/25%    1         10        1          6h1m
frontend   Deployment/frontend   51%/25%   1         10        1          6h1m
frontend   Deployment/frontend   51%/25%   1         10        3          6h2m
frontend   Deployment/frontend   65%/25%   1         10        3          6h2m
```
*RESULT :* Initially the number of replicas were 1, and as Load goes beyond 25%, HPA scaled more pods. In this case Load was around 65% and Number of replicas were 3.

- Now go back to your previous terminal and stop load-generator pod.

- Again watch autoscaler output. After some time, you will see load is getting normalized and number of replicas going donw. Something like this :
```
kubectl get hpa frontend -n staging -w
```
```
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
frontend   Deployment/frontend   43%/25%   1         10        3          6h3m
frontend   Deployment/frontend   43%/25%   1         10        6          6h3m
frontend   Deployment/frontend   1%/25%    1         10        6          6h3m
frontend   Deployment/frontend   1%/25%    1         10        6          6h3m
frontend   Deployment/frontend   1%/25%    1         10        6          6h8m
frontend   Deployment/frontend   1%/25%    1         10        1          6h8m
```
*RESULT :* Number of replicas back to 1
