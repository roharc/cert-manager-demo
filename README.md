# How To: setup step CA and integrate into kubernetes with cert-manager  
## 1. Setup CA and ACME provider with step CA  

### Download and install packages  
https://smallstep.com/docs/step-ca/installation/  

```
wget https://dl.smallstep.com/cli/docs-ca-install/latest/step-cli_amd64.deb
sudo dpkg -i step-cli_amd64.deb

wget https://dl.smallstep.com/certificates/docs-ca-install/latest/step-ca_amd64.deb
sudo dpkg -i step-ca_amd64.deb
```

### Setup service user   

https://smallstep.com/docs/step-ca/certificate-authority-server-production/#create-a-service-user-to-run-step-ca

```
sudo useradd --system --home /etc/step-ca --shell /bin/false step
```  

### Grant capability to bind portnumbers less than 1024 to step-ca binary

```
sudo setcap CAP_NET_BIND_SERVICE=+eip $(which step-ca)
```  

### Initialize CA

```
step ca init
```

```
✔ Deployment Type: Standalone
What would you like to name your new PKI?
✔ (e.g. Smallstep): pki-demo
What DNS names or IP addresses will clients use to reach your CA?
✔ (e.g. ca.example.com[,10.1.2.3,etc.]): pki-demo,pki-demo.home.arpa,192.168.0.99,127.0.0.1
What IP and port will your new CA bind to? (:443 will bind to 0.0.0.0:443)
✔ (e.g. :443 or 127.0.0.1:443): :443
What would you like to name the CA's first provisioner?
✔ (e.g. you@smallstep.com): first
Choose a password for your CA keys and first provisioner.
✔ [leave empty and we'll generate one]:
```

### Move step-ca to `/etc/step-ca` and change ownership

```
sudo mkdir /etc/step-ca
sudo mv ~/.step/* /etc/step-ca
sudo chown -R step:step /etc/step-ca
```

### Modify paths in config files

```
sudo sed 's/home\/xxx\/.step/etc\/step-ca/g' /etc/step-ca/config/ca.json
sudo sed 's/home\/xxx\/.step/etc\/step-ca/g' /etc/step-ca/config/defaults.json
```
(use `sed -i` to actually write the files after checking)

### Create passwordfile and change ownership

```
sudo vi /etc/step-ca/password.txt
```
```
sudo chown step:step /etc/step-ca/password.txt
```

### Create unitfile, start and check service

https://smallstep.com/docs/step-ca/certificate-authority-server-production/#running-step-ca-as-a-daemon
```
sudo vi  /etc/systemd/system/step-ca.service
```
```
sudo systemctl enable --now step-ca && sudo systemctl status step-ca
```

### Add ACME provisioner and restart service

```
sudo STEPPATH=/etc/step-ca step ca provisioner add acme --type acme --ca-url https://127.0.0.1 --root /etc/step-ca/certs/root_ca.crt
```
```
sudo systemctl restart step-ca
```
```
sudo STEPPATH=/etc/step-ca step ca provisioner list --ca-url https://127.0.0.1
```

### fetch CA Bundle for k8s integration

```
sudo cat /etc/step-ca/certs/intermediate_ca.crt /etc/step-ca/certs/root_ca.crt > ca_bundle.crt && chown xxx:xxx ca_bundle.crt
```

## 2. Integrate PKI into kubernetes with cert-manager

prerequisites: Kubernetes Node with helm installed

### scp CA Bundle from pki node

```
scp pki-demo:ca_bundle.crt .
```

### try to curl CA

```
curl https://pki-demo.home.arpa
```

### add CA to system trust store

```
sudo cp ca_bundle.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates
```

### curl CA once more to confirm trust

```
curl https://pki-demo.home.arpa
```

### install step-cli and request cert as a test

```
wget https://dl.smallstep.com/cli/docs-ca-install/latest/step-cli_amd64.deb
sudo dpkg -i step-cli_amd64.deb
```
```
sudo step ca certificate k3s-demo.home.arpa k3s-demo.crt k3s-demo.key --acme https://pki-demo.home.arpa/acme/acme/directory --san k3s-demo.home.arpa --san k3s-demo --san 192.168.0.98
```








