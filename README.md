# k8s-test

## Professional Test
#### Prerequisites : 
- Development instance 
- GCP Account
- Setup gcloud command-line on your development box
  - For Debian/Ubuntu : https://cloud.google.com/sdk/docs/quickstart-debian-ubuntu
  - For macOS : https://cloud.google.com/sdk/docs/quickstart-macos
  - For RedHat and CentOS : https://cloud.google.com/sdk/docs/quickstart-redhat-centos
- ubuntu user with passwordless sudo access on your development box
- Public internet access
- Docker Hub Account <Account Details Given In Mail>
- Github account <Account Details Given In Mail>

#### Defaults Taken
- Kubernetes
  - Region : asia-south1
  - Zone : asia-south1-c
  - Network Name : mstakx-test
  - Master Node Count : 2 (Free tier only supports max 8 CPUS in a region, so 4 VM's each with 2 vcpus)
  - Worker Node Count : 2
- Jenkins
  - Region : asia-southeast1-b

#### First Step
- clone this project and cd into repo
```
git clone https://github.com/taherbohari/k8s-test.git
cd k8s-test
```

#### Setup Local Instance
- Run install-prereq script
```
sudo ./professional/install-prereq
```
#### Setup Kubernetes
- Bringup k8s vms on gcloud
```
./professional/k8s/bringup-k8s-cluster
```
- ssh to controller-0 instance. This is just to fetch gcloud ssh keys on your local instance.
```
gcloud compute ssh controller-0
```
- Exit from controller-0

- Run k8s prereq
```
./professional/k8s/setup-k8s
```
- Create kubeadm config
```
./professional/k8s/create-kubeadm-config
```
##### Setup controller-0 instance. Below steps to run on controller-0 instance
- ssh to controller-0 instance
```
gcloud compute ssh controller-0
```
- Initialize kubeadm on controller-0 instance. This step will take couple of minutes to run
```
sudo kubeadm init --config=kubeadm.yaml --experimental-upload-certs
```
**NOTE : Save the console output of this command on your local instance.**
- Setup k8s for ubuntu user on controller-0 instance
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- Apply the CNI plugin
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
- Exit from controller-0 instance

##### Setup remaining controller instances

**NOTE :** Run below step on all remaining controller nodes
- Now ssh to controller-1 instance and add them to cluster.
```
gcloud compute ssh controller-1
```
Run below command as root. Refer the console output saved from above step
```
sudo kubeadm join <CONTROL_PLANE_ENDPOINT> --token <TOKEN> \
    --discovery-token-ca-cert-hash <CERT> \
    --experimental-control-plane --certificate-key <CERT>
```
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
##### Setup remaining worker instances

**NOTE :** Run below step on all worker nodes
- Now ssh to both worker nodes : worker-0, worker-1 and add them to cluster.
```
gcloud compute ssh worker-0
gcloud compute ssh worker-1
```
Run below command as root. Refer the console output saved from above step

**NOTE :** Do not include *--experimental-control-plane*  option for worker nodes 
```
sudo kubeadm join <CONTROL_PLANE_ENDPOINT> --token <TOKEN> \
    --discovery-token-ca-cert-hash <CERT>
```
- Once all nodes have joined the cluster. Exit from worker nodes and check the status of nodes on controller-0 instance
```
gcloud compute ssh controller-0
kubectl get nodes
exit
```

###### Setup your local instance
- Add remaining controller instance to the kubernetes target pool
```
gcloud compute target-pools add-instances kubernetes-target-pool --instances controller-1
```
- Setup your local instance to talk to your kubernetes cluster
```
./professional/setup-local-k8s-access
```
- Deploy guestbook application. This step will create development namespace and deploy guestbook app in the same.
```
./professional/k8s/deploy-guestbook-app
```
- Deploy nginx app which will be used in later part (jenkins)
```
./professional/k8s/deploy-nginx-app
```
- Test nginx app. Check message.
```
./professional/k8s/test-nginx-app
```
- Setup helm on local instance
```
./professional/k8s/setup-helm
```
#### Setup Jenkins
- Bringup jenkins instance on gcloud
```
./professional/jenkins/bringup-jenkins
```
- Install Jenkins prereq
```
./professional/jenkins/setup-jenkins
```
- Login to jenkins now. Pick jenknis url from console output of last command.

*NOTE :* Follow the steps on UI to login and install default plugins on jenkins.

- Get the jenkins password from file location present on jenkins instance
```
gcloud compute ssh jenkins --zone=asia-southeast1-b
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- Configure Jenkins.
	- Access to All for importing jobs via CLI
	  - Manage Jenkins -> Configure Global Security -> Authorization -> Anyone can do anything (check this radio button)
	- Add Environment Variable named *DOCKER_HUB* with Docker account Password as value. Password given in *EMAIL*.
	  - Manage Jenkins -> Configure System -> Global properties -> Environment Variables
- Import jobs into jenkins. Run below command on your Local Instance
```
./professional/jenkins/import-jobs
```
- Access **k8sadmin** Github account. Access details provided in *EMAIL*.
	- Update webhook of k8sadmin/custom_nginx project
	- Go to Project Settings -> WebHook -> Add Webhook
```
	#Command to get jenkins public IP
	gcloud compute instances describe jenkins --zone=asia-southeast1-b --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```
	- *Payload URL :* http://<JENKINS_PUBLIC_IP>:8080/github-webhook/
	- *Content Type :* application/x-www-form-urlencoded
	- *Secret : * Jenkins Secret of first admin user created by you
- Commands to test CI/CD Pipeline using jenkins
	- Clone custom_nginx repo and update index.html
  ```
  git clone https://github.com/k8sadmin/custom_nginx.git
  cd custom_nginx
  ## update message string inside 'index.html' file.
  ## commit and push the contents to repo with provided creds
  ```
  *RESULT:* This will trigger job *mynginx* which will then build new docker image, push it into docker hub and deploy new image into our kubernetes cluster

  - Test nginx app. Now app should show updated message.
  ```
  ./professional/k8s/test-nginx-app
  ```
- Now run hello-world-helm jenkins job to deploy helm app in k8s

#### Setup Monitoring
- Setup monitoring
```
./professional/monitoring/setup-monitoring
```
- Login to Grafana. 

*NOTE* : Check console output of above command for URL and Credentials.
  - In the left hand menu, Click on "+" -> Import
  - In the Grafana.com dashboard input, add the dashboard ID we want to use ie. **1860** and click Load
  - On the next screen select a name for your dashboard and select Prometheus as the datasource for it and click Import.

#### Canary Deployment
- Deploy hello world app
```
./professional/canary/deploy-hello-world
```
- deploy hello world canary
```
./professional/canary/deploy-hello-world-canary
```
- check canary deployments
```
./professional/canary/run-hello-world-svc
```
*RESULT :* You will see both versions populated of hello-world app

#### Blue Green Deployment
- Deploy color-app with blue color in blue namespace
```
./professional/blue-green/deploy-blue
```
- Access blue color-app on above url. Check console output of above command.

*RESULT :* You should see the BLUE color screen

- Deploy color-app with green color in green namespace
```
./professional/blue-green/deploy-green
```
- Access green color-app on above url. Check console output of above command.

*RESULT :* You should see the GREEN color screen

- Update color app deployment
```
./professional/blue-green/update-blue-to-green
```
- Access color-app using above url. Check console output of above command.

*RESULT :* Initially there was BLUE color screen on this URL, now you should see the GREEN color screen.
