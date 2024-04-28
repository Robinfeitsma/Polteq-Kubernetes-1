# Kubernetes 1


## Setup virtualbox

- [ ] Download en installeer virtualbox 7.0.14: 
- https://www.virtualbox.org/wiki/Download_Old_Builds_7_0


- [ ] Download Images (Master & Node 1)
- https://polteq-my.sharepoint.com/:f:/p/robin_feitsma/EjQOYpa70HhHghZp0bCaYXYB2ZjTEgU6yLdisHQMSVfrAQ?e=JZf5wu

- [ ] Start virtualbox, Ga naar File en klik op import appliance

- [ ] Selecteer en importeer beide gedownloaden images (gebruik standaard instellingen)

- [ ] Ga naar File ->Tools -> en klik op Network manager

- [ ] Klik op Host only Networks -> Create

- [ ] Klik op Configure adaptor manually en zet ip= 192.168.56.1

## Install k3s

- [ ] SSH naar master en install k3s met de volgende commando

````
curl -sfL https://get.k3s.io | sh -s - --node-taint CriticalAddonsOnly=true:NoExecute  --disable-cloud-controller
````

- [ ] Haal de master token op 

````
sudo cat /var/lib/rancher/k3s/server/node-token
````

- [ ] Kopieer de master token (selecteer in powershell en klik rechts met muis)

- [ ] SSH naar node 1 (ssh user@192.168.56.4) en install k3s met de volgende commando

````
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.56.3:6443 K3S_TOKEN=mynodetoken sh -
````

- [ ] Ga terug naar de master node 

````
exit
````

- [ ] Controleer of alle nodes zijn toegevoegd

````
sudo kubectl get nodes
````

- [ ] Zet de k3s configuratie in de environments(zowel master als node)

````
sudo echo "KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> /etc/environment
````

## Installeer Helm

````
#Fix kubeconfig file
export KUBECONFIG=~/.kube/config
mkdir ~/.kube 2> /dev/null
sudo k3s kubectl config view --raw > "$KUBECONFIG"
chmod 600 "$KUBECONFIG"
sudo su
echo "KUBECONFIG=$KUBECONFIG" >> /etc/environment
exit
#Switch naar home
cd
#Maak een directory voor helm
mkdir helm
#Switch naar helm directory
cd helm
#Download helm installer
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
#Verander rechten voor uitvoering van bestanden
chmod 700 get_helm.sh
#installeer helm
./get_helm.sh
#check of helm is geinstalleerd
helm version
````

## Installeer configureer MetalLB

````
# Voeg metallb repository toe aan helm
helm repo add metallb https://metallb.github.io/metallb
# Controleer of dat goed is gegaan
helm search repo metallb
# Installeer metallb
helm upgrade --install metallb metallb/metallb --create-namespace \
--namespace metallb-system --wait
````

- [ ] Voeg de configuratie toe

````
cat << 'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.56.200-192.168.56.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF
````

- [ ] Bekijk de pods (binnen namespace metallb-system)

````
sudo kubectl get pods -n metallb-system
````
- [ ] Bekijk de services (binnen namespace metallb-system)

````
sudo kubectl get svc -n kube-system
````

## Installeer configureer Argo-CD

````
sudo su
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
````

- [ ] Bekijk de pods en wacht tot ze "Running" zijn

````
kubectl get pods  -n argocd
````

- [ ] Bekijk de services 

````
kubectl get svc -n argocd
````

- [ ] Patch Argo-cd met een extern ip adres

````
kubectl patch service argocd-server -n argocd --patch '{ "spec": { "type": "LoadBalancer", "loadBalancerIP": "192.168.56.208" } }'
````

- [ ] Open je browser en voer in https://192.168.56.208/login

- [ ] Ga naar powershell en haal je wachtwoord op

````
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
````

- [ ] Ga naar de browser en login als username: admin wachtwoord:<wachtwoord uit powershell>

- [ ] https://github.com/settings/tokens Maak een token aan en zet die in je wachtwoord

