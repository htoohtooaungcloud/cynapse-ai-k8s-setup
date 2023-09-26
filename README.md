# Kubernetes Managed Applications and Observability Task
1. Install Kubernetes locally on your laptop with the distribution of your choice or feel free to use the cloud environment as well.
2. Install a sample application (of your choice) that connects to a database that is managed by your Kubernetes cluster
3. Push the logs into Prometheus/Loki and show us that you are able to display metrics nicely in Grafana

# Preposed architecture
![kubernetes-aws-observability](https://github.com/htoohtooaungcloud/cynapse-ai-k8s-setup/assets/54118047/46aa58ee-0196-43ae-9475-45d4c8b05e60)

## Taks list
1. Build the infrastructure using terraform and several EC2 Instances in AWS "ap-southeast-1" + VPC + Subnets + IGW + KeyPair + SecuritGroups and others necessary resources.
2. Create the ansible-terraform provider resouces for ENVIRONMENT VARIABILE before carry out configuration changes using Ansible.
3. terraform apply and build the infrastructer
4. Once we done deploying infrastructure in AWS, execute "ansible-playbook -i <file>". (Some of the stages has problem and need to fixed. Therefore, stick to manual stepup in each server)
5. Run the "common.sh", "master.sh" and join to the master-node from worker-nodes to form kubernetes cluster using Kubeadm Bootstrap setup.
6. We're going to use cilium as CNI and disable the kube-proxy. burdens because it eBPF features. 
  - (Basically, it allows to run sandbox programs in the Linux Kernel without going back and forth between Kernel space and userspace which is what iptables do)
7. We'll be using cilium as loadbalancer instead of metallb as well. However, we still need to create AWS Elastic-LoadBalancer and point to the kubernets api server endpoint 6443 to expose service as LoadBalancer. 
  - This is need to be done and compulsory, but as for now let's stick to NodePort when we expose the kubernetes services.
8. Deploy the applications using kubectl command (Mongo-frontend app,  Mongodb app). Using Deployment for frontend-mongo express app and StatefulSet for mongo-db app.
9. Deploy the observability applicaitons using helm (Prometheus, Grafana and Grafana Loki). 
  - **Note the (1) kube-prometheus-stack and loki-stack helm charts should be installed.**
10. Expose the service for Prometheus, Grafana, Mongo-express. In this case, I'm using NodePort.
11. Login via Web access and explore how to display the metrics and logs from Grafana
11. Create GitHub repository as a version control system for GitOps (Singel source of truth, Continuous reconciliation)
12. Install argocd for GitOps workflow.

## Technology and tech stack for this task

* *Terraform* (Infrastructure as Code)
* *Ansible* (Configuration Management)
* *Kubernetes* (Container Orchestation)
* *Cilium* (Container Network Interface)
* *Grafana* (Visualization)
* *Premetheus* (Metrics)
* *Grafana Loki* (Logging)
* *GitHub* (Version Control System)
* *ArgoCD* (GitOps)
* *Mongo-express & Mongo-db* (Business Application)

### Create private-key.pem file with write permisssion only after terraform apply in you vscode directory
```
touch private-key.pem
sudo chmod 600 private-key.pem
```
### Let's use Ansible to make changes on each server. If any errors occur, manual SSH into the server to make the necessary changes
```
ansible-playbook -i inventory.yml playbook.yml
```

### Steps to do after provision
### SSH command to login to server
```
ssh -i private-key.pem ubuntu@54.169.18.228
ssh -i private-key.pem ubuntu@13.212.139.199
ssh -i private-key.pem ubuntu@13.229.211.89
```

### Change the permission of the scripts file in each server then execute
```
sudo chmod +x *.sh
bash common.sh # for all nodes
bash master.sh # only for master
```

--------------------
### Testing Phase
--------------------
```
sudo kubectl --kubeconfig /etc/kubernetes/admin.conf get pods -A
``` 


### To make kubectl work for your non-root user
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

--------------------------------------------
## Join to master-node01 from worker nodes to form kubernetes cluster
--------------------------------------------
```
sudo kubeadm join <public-ip-master-node>:6443 --token njimf1.pxbir7lvm7w7qd6s \
	--discovery-token-ca-cert-hash sha256:4613fa29c8ab29a23d254b1ceff7ebe2f3765f1b29d7ba0fa1633ace6c7f9ce1
```


## Set some host configure in each server 
```
sudo hostnamectl set-hostname master-node01
sudo echo "<public-ip-master-node>  master-node01  master-node01.cynapse.io" >> /etc/hosts
sudo echo "172-31-39-238  master-node01  master-node01.cynapse.io" >> /etc/hosts
sudo echo "<public-ip-worker-node01> worker-node01  worker-node01.cynapse.io" >> /etc/hosts
sudo echo "172.31.43.65 worker-node01  worker-node01.cynapse.io" >> /etc/hosts
sudo echo "<public-ip-worker-node02> worker-node02  worker-node01.cynapse.io" >> /etc/hosts
sudo echo  "172-31-41-62  worker-node02  worker-node02.cynapse.io" >> /etc/hosts

sudo hostnamectl set-hostname worker-node01
sudo echo "<public-ip-master-node>  master-node01  master-node01.cynapse.io" >> /etc/hosts
sudo echo "172-31-39-238  master-node01  master-node01.cynapse.io" >> /etc/hosts
sudo echo "<public-ip-worker-node01> worker-node01  worker-node01.cynapse.io" >> /etc/hosts
sudo echo "172.31.43.65 worker-node01  worker-node01.cynapse.io" >> /etc/hosts
sudo echo "<public-ip-worker-node02> worker-node02  worker-node01.cynapse.io" >> /etc/hosts
sudo echo  "172-31-41-62  worker-node02  worker-node02.cynapse.io" >> /etc/hosts

sudo hostnamectl set-hostname worker-node02
sudo echo "<public-ip-master-node>  master-node01  master-node01.cynapse.io" >> /etc/hosts
sudo echo "172-31-39-238  master-node01  master-node01.cynapse.io" >> /etc/hosts
sudo echo "<public-ip-worker-node01> worker-node01  worker-node01.cynapse.io" >> /etc/hosts
sudo echo "172.31.43.65 worker-node01  worker-node01.cynapse.io" >> /etc/hosts
sudo echo "<public-ip-worker-node02> worker-node02  worker-node01.cynapse.io" >> /etc/hosts
sudo echo  "172-31-41-62  worker-node02  worker-node02.cynapse.io" >> /etc/hosts
```


# test the svc with cilium


## Preperation for install longhorn
```
sudo vi /etc/multipath.conf

blacklist {
    devnode "^sd[a-z0-9]+"
}

sudo systemctl restart multipathd.service
sudo multipath -t
```

### Install jq
```
sudo apt install jq -y
```

### Download the script to check env to install longhorn (only for master-node01) 
### Link [https://medium.com/@ramkicse/how-to-install-longhorn-distributed-block-storage-system-for-kubernetes-811f8afc4d8e]
```
wget https://raw.githubusercontent.com/longhorn/longhorn/v1.3.0/scripts/environment_check.sh
chmod +x environment_check.sh
./environment_check.sh
```

### rectify the errors for nfs-common (all nodes)
```
sudo apt install nfs-common -y
sudo systemctl status iscsid
sudo systemctl restart iscsid
sudo systemctl enable iscsid
sudo systemctl status iscsid
```

### test and verity again in worker-nodes
```
./environment_check.sh
```

### Mongo applications deploymemnt through kubectl command in frontend-backend-mongo
please go the fronedend-backend-mongo app and deploy the necessary yaml file with "kubectl apply -f" 
please creat configmap and secret yaml at first
```
kubectl apply -f mongo-cm.yaml
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo-express.yaml
kubectl apply -f mongodb-statefulset.yaml
```

### Different Monitoring Namespace for Prometheus Operator (kube-prometheus-stack) 
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus | grep -i kube-prometheus-stack 
```

### Verify
NAME                                              	CHART VERSION	APP VERSION	DESCRIPTION                                       
prometheus-community/kube-prometheus-stack        	50.0.0       	v0.67.1    	kube-prometheus-stack collects Kubernetes manif...
helm install kube-prometheus prometheus-community/kube-prometheus-stack -n monitoring
### Customize dashboard can be downloaded from here 
[https://grafana.com/grafana/dashboards/]
[https://github.com/prometheus-community/helm-charts/blob/kube-prometheus-stack-51.2.0/charts/kube-prometheus-stack/values.yaml]
[https://sysdig.com/blog/prometheus-query-examples/]


## Loki installation setup
```
helm search repo loki # we are going to use loki-stack
helm show values grafana/loki-stack > values-1.yaml 
helm install --values values.yaml loki --namespace monitoring grafana/loki-stack
hem list -A
```

## check the secret of kube-prometheus-grafana application to login thruough webpage
```
kubectl get secret kube-prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode 
kubectl get secret loki-promtail -n monitoring -o jsonpath="{.data.promtail\.yaml}" | base64 --decode
```
> [!NOTE]
> Add in grafana new connection http://10-0-2-150.loki.monitoring.svc.cluster.local:3100 to connect to loki-0 pod

### Advanced loki testing
```
kubectl get secret loki-promtail -n monitoring -o jsonpath="{.data.promtail\.yaml}" | base64 --decode > promtail.yaml
```

Modification
> Delete the existing secret loki-promtail and apply new secret --from-file=./promtail.yaml
> Delete all pods "kubectl delete pod <pod-name> -n monitoring

### Git setup for GitOps
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/cynapse-ai
ssh -T git@github.com
```

### Create git repo via api but msut have to export api key prior
```
curl -L \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer ghp_vU5rX6ROV48o4k0KP0aejmhSmQxxxxxxxxx" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/user/repos \
  -d '{"name":"cynapse-ai-k8s-setup","description":"This is my repo!","homepage":"https://github.com","private":false,"is_template":true}'
```

### Export argocd values.yaml file from helm chart
```
helm show values argo/argocd-apps > values.yaml
```

### Install argocd  
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml
kubectl edit service/argocd-server -n argocd # to NodePort
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode; echo # check the argocd ui login password
kubectl apply -f mongo-argo-secret-pw.yaml # secret must create first
kubectl apply -f mongo-argo/mongo-argocd-app.yaml
```