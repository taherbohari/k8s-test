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
./begginer/install-prereq
```

- Export all required parameters
```
export PROJECT_ID=<provide your own project_id>
export K8S_CLUSTER_NAME="test-cluster"
export K8S_REGION="asia-south1"
export NODES_PER_ZONE="2"
export MACHINE_TYPE="n1-standard-1"
export K8S_VER="1.12.8-gke.6" <This version should be supported by GKE>
```
**NOTE : ** Update values as per your requirement

- Create your first gcloud project
```
gcloud projects create ${PROJECT_ID}
```

- Create kubernetes cluster
```
./begginer/setup-gcloud-k8s-cluster
```

- Setup helm
```
./beginner/setup-helm
```

- Deploy ingress controller
```
./begginer/deploy-ingress-controller
```

- Deploy guestbook application
```
./beginner/setup-guestbook-application
```

- Deploy Horizontal Pod Autoscaler
```
./begginer/deploy-hpa
```
