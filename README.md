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
   ```

2. **Haal de master token op**:
   ```sh
   sudo cat /var/lib/rancher/k3s/server/node-token
   ```
    - Kopieer de master token (selecteer in PowerShell en klik rechts met de muis)

3. **SSH naar Node 1 en installeer k3s**:
   ```sh
   ssh user@192.168.56.4
   curl -sfL https://get.k3s.io | K3S_URL=https://192.168.56.3:6443 K3S_TOKEN=<master-token> sh -
   ```

4. **Ga terug naar de Master Node**:
   ```sh
   exit
   ```

5. **Controleer of alle nodes zijn toegevoegd**:
   ```sh
   sudo kubectl get nodes
   ```
   - Als er problemen zijn kan je op de node dit commando gebruiken om de status van de k3s-agent te bekijken:
         
   - ```sh
         sudo systemctl status k3s-agent.service

6. **Zet de k3s configuratie in de environment (zowel Master als Node)**:
   ```sh
   sudo su
   echo "KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> /etc/environment
   ```
7. **Reboot de Master node**:
   ```sh
   reboot
   ```

## Installeer Helm

1. **Fix kubeconfig file**:
   ```sh
   export KUBECONFIG=~/.kube/config
   mkdir ~/.kube 2> /dev/null
   k3s kubectl config view --raw > "$KUBECONFIG"
   ```

2. **Installeer Helm**:
    - Ga naar home directory:
      ```sh
      cd
      ```
    - Maak een directory voor Helm:
      ```sh
      mkdir helm
      cd helm
      ```
    - Download Helm installer:
      ```sh
      curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
      chmod 700 get_helm.sh
      ./get_helm.sh
      ```
    - Controleer Helm installatie:
      ```sh
      helm version
      ```

## Installeer en configureer MetalLB

1. **Voeg MetalLB repository toe aan Helm**:
   ```sh
   helm repo add metallb https://metallb.github.io/metallb
   helm search repo metallb
   ```

2. **Installeer MetalLB**:
   ```sh
   helm upgrade --install metallb metallb/metallb --create-namespace --namespace metallb-system --wait
   ```

3. **Voeg de configuratie toe**:
   ```sh
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
   ```

4. **Bekijk de pods in MetalLB-system namespace**:
   ```sh
   sudo kubectl get pods -n metallb-system
   ```

5. **Bekijk de services in kube-system namespace**:
   ```sh
   sudo kubectl get svc -n kube-system
   ```

## Installeer en configureer Argo-CD

1. **Installeer Argo-CD**:
   ```sh
   sudo su
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. **Bekijk de pods en wacht tot ze "Running" zijn**:
   ```sh
   kubectl get pods -n argocd
   ```

3. **Bekijk de services**:
   ```sh
   kubectl get svc -n argocd
   ```

4. **Patch Argo-CD met een extern IP-adres**:
   ```sh
   kubectl patch service argocd-server -n argocd --patch '{ "spec": { "type": "LoadBalancer", "loadBalancerIP": "192.168.56.208" } }'
   ```

5. **Open je browser en ga naar**: [https://192.168.56.208/login](https://192.168.56.208/login)

6. **Haal je wachtwoord op via PowerShell**:
   ```sh
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
   ```

7. **Login bij Argo-CD**:
    - Gebruikersnaam: `admin`
    - Wachtwoord: `<wachtwoord uit PowerShell>`

8. **Maak een GitHub token aan en gebruik deze als wachtwoord**:  
   [GitHub Token Aanmaken](https://github.com/settings/tokens)

## Koppel repository 

1. **Ga naar Settings -> Repositories**
2. **Klik op + Connect REPO**
3. **Vul in**:
   - Choose your connection method: VIA HTTPS
   - Type: git
   - Project: default
   - Repository URL: https://github.com/Robinfeitsma/Polteq-Kubernetes-1
   - Username: Test (of een andere string)
   - Password: Je GitToken
4. **Klik op connect**