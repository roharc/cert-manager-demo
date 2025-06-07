# How To: setup step CA and integrate into kubernetes with cert-manager  
## Setup CA with step CA  

### Download and install packages  
https://smallstep.com/docs/step-ca/installation/  

```
wget https://dl.smallstep.com/cli/docs-ca-install/latest/step-cli_amd64.deb
sudo dpkg -i step-cli_amd64.deb

wget https://dl.smallstep.com/certificates/docs-ca-install/latest/step-ca_amd64.deb
sudo dpkg -i step-ca_amd64.deb
```

### Setup system user   

https://smallstep.com/docs/step-ca/certificate-authority-server-production/#create-a-service-user-to-run-step-ca

`sudo useradd --system --home /etc/step-ca --shell /bin/false step`  

### grant capabilty to bind portnumbers less than 1024 to step-ca binary

`sudo setcap CAP_NET_BIND_SERVICE=+eip $(which step-ca)`  

### initialize CA

```
$ step ca init
✔ Deployment Type: Standalone
What would you like to name your new PKI?
✔ (e.g. Smallstep): pki-demo
What DNS names or IP addresses will clients use to reach your CA?
✔ (e.g. ca.example.com[,10.1.2.3,etc.]): pki-demo.home.arpa
What IP and port will your new CA bind to? (:443 will bind to 0.0.0.0:443)
✔ (e.g. :443 or 127.0.0.1:443): :443
What would you like to name the CA's first provisioner?
✔ (e.g. you@smallstep.com): first
Choose a password for your CA keys and first provisioner.
✔ [leave empty and we'll generate one]:
```

### move step-ca to `/etc/step-ca` and change ownership

```
sudo mkdir /etc/step-ca
sudo mv ~/.step/* /etc/step-ca
sudo chown -R step:step /etc/step-ca
```

### modify paths in config files

```
sudo sed 's/home\/xxx\/.step/etc\/step-ca/g' /etc/step-ca/config/ca.json
sudo sed 's/home\/xxx\/.step/etc\/step-ca/g' /etc/step-ca/config/defaults.json
```
(use `sed -i` to actually write the files after checking)
### create unitfile

https://smallstep.com/docs/step-ca/certificate-authority-server-production/#running-step-ca-as-a-daemon
```
sudo vi  /etc/systemd/system/step-ca.service
sudo systemctl enable --now step-ca
sudo systemctl status step-ca
```

### create passwordfile

```
sudo vi /etc/step-ca/password.txt
sudo chown step:step /etc/step-ca/password.txt
```

### add ACME provisioner and restart (reload?) service

```
sudo STEPPATH=/etc/step-ca step ca provisioner add acme --type acme --ca-url https://pki-demo.home.arpa --root /etc/step-ca/certs/root_ca.crt
systemctl reload step-ca
```

## integrate PKI into kubernetes with cert-manager





