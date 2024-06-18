# Kubernetes 1

## Setup VirtualBox

1. **Download en installeer VirtualBox 7.0.14**:  
   [VirtualBox 7.0.14 Download](https://www.virtualbox.org/wiki/Download_Old_Builds_7_0)

2. **Download Images (Master & Node 1)**:  
   [Images Download Link](https://polteq-my.sharepoint.com/:f:/p/robin_feitsma/EjQOYpa70HhHghZp0bCaYXYB2ZjTEgU6yLdisHQMSVfrAQ?e=JZf5wu)

3. **Start VirtualBox en importeer Images**:
    - Ga naar `File` > `Import Appliance`
    - Selecteer en importeer beide gedownloade images (gebruik standaard instellingen)

### Setup VirtualBox Network

4. **Configureer Netwerk**:
    - Ga naar `File` > `Tools` > `Network Manager`
    - Klik op `Host-only Networks` > `Create`
    - Configureer adapter handmatig en zet IP op `192.168.56.1`

### Setup VirtualBox Network voor Master

5. **Configureer netwerk voor Master**:
    - Ga naar `Master` > `Settings` > `Network`
    - Controleer of `Adapter 1` is ingesteld op `Bridged Adapter`
    - Stel `Adapter 2` in op `Host-only Network`
    - Ga naar `Advanced` bij `Adapter 2` en zet `Promiscuous Mode` op `Allow VMs`

### Setup VirtualBox Network voor Node 1

6. **Configureer netwerk voor Node 1**:
    - Ga naar `Node` > `Settings` > `Network`
    - Controleer of `Adapter 2` is ingesteld op `Bridged Adapter`
    - Stel `Adapter 1` in op `Host-only Network`
    - Ga naar `Advanced` bij `Adapter 1` en zet `Promiscuous Mode` op `Allow VMs`

## Install k3s

1. **SSH naar Master en installeer k3s**:
   ```sh
   curl -sfL https://get.k3s.io | sh -s - --node-taint CriticalAddonsOnly=true:NoExecute --disable-cloud-controller
   
2. **Haal de master token op**:
   ```sh
   sudo cat /var/lib/rancher/k3s/server/node-token

3. **Kopieer de master token**:
   -  Selecteer in powershell en klik rechts met muis

4. **SSH naar node 1**:
   ```sh
   ssh user@192.168.56.4

5. **install k3s met de volgende commando**:
   ```sh
   curl -sfL https://get.k3s.io | K3S_URL=https://192.168.56.3:6443 K3S_TOKEN=<mynodetoken> sh -

6. **Ga terug naar de master node met commando**:
   ```sh
   exit

7. **Controleer of alle nodes zijn toegevoegd**:
   ```sh
   sudo kubectl get nodes

8. **Zet de k3s configuratie in de environments**:
- Zowel master als node

   ```sh
   sudo su
   echo "KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> /etc/environment
   exit

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
cd..
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

