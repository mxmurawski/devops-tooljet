devops-demo-tooljet:microk8s repositroy presents tooljet application stack deployment from scratch to on-premises kubernetes cluster using helm

1. Install Kubernetes on Ubuntu machine:

snap install --classic microk8s

2. Create kube config in your home directory:

mkdir ~/.kube
microk8s.config > ~/.kube/config

3. Enable metal load balancer to easily access your internal kubernetes resources. You can specify a single or a pool of IP addresses to be used by metal load balancer. In my case I want just one load balancer to be attached to my cluster:

microk8s enable metallb:10.200.0.33

4. Enable nginx ingress controller to route network traffic to proper services.

microk8s enable ingress

5. Create LoadBalancer object in your Kubernetes cluster and bind ports 80 and 443 to it.

kubectl apply -f service-loadbalancer.yaml

6. To be able to serve the application over encrypted channel - https, create secret object with fullchain ssl certificate and corresponding private key. To do that, create cert.pem and key.pem files in current directory and fill those files with proper content. Remember! In fullchain certificate file the proper order of certificates is as follows:
    
    1. Server certificate
    2. Intermediate CA certificate (optional)
    3. Root CA certificate


kubectl create secret tls ingress-tls --cert=cert.pem --key=key.pem --namespace ingress

7. We would like our cluster to serve applications in domain k8s.ad.mikolajmurawski.pl. To do that, add an A wildcard record to your DNS server pointing to the load balancer's ip address. In my scenario the address is 10.200.0.33. Then create general ingress responsible for redirecting users from http 80 to http 443, encrypting network traffic using created tls secret and forward request for the next, more specific ingress host match:

kubectl apply -f ingress.yaml

8. Tooljet application stack contains postgresql and redis servers deployment. To persist their data we need to provide kubernetes cluster an extrernal storage provider. I chose to use my NAS server accessible at nas.ad.mikolajmurawski.pl:/mnt/pool/nfs. To be able to create persistent volumes on NAS server we need to install container storage interface for nas:

helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system  --set kubeletDir=/var/snap/microk8s/common/var/lib/kubelet

9. Create storage provider object allowing to connect to the NAS server using its address and the share's mount point.

kubectl apply -f sc-nfs.yaml

10. Finally we are ready to deploy the Tooljet application stack to kubernetes cluster. Copy tooljet/values.yaml file to current directory and fill it with proper values. You can take pattern from the example-values.yaml file.

helm install tooljet .\tooljet --values .\values.yaml