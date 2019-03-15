ANSIBLE DEPLOYMENT HOME PI CLUSTER
==================================

### Enable password less authtentication.
#	Log in all the Pis via ssh, accept the keys. ( erase the old ones)
#	Copy ssh keys:
	ssh-copy-id pi@pi3

### Disable Host-Key authentication on ansible.cfg

## REMEMBER TO DO THIS:
	ansible all -m ping --ask-pass

### ENABLE CGROUPS
# ADD in /boot/cmdline.txt , before elevator=deadline : cgroup_enable=memory cgroup_memory=1 

  sed -i 's/\<elevator=deadline\>/cgroup_enable=memory cgroup_memory=1 &/' /boot/cmdline.txt

### For a static IP address on an Ethernet connection:

#  sudo nano /etc/dhcpcd.conf.
#   Type in the following lines on the top of the file: 

    interface eth0 static ip_address=192.168.8.12/24 static routers=192.168.8.1 static domain_name_servers=192.168.8.1
    sudo reboot

###  DEPLOY WEAVE BEFORE JOINING WORKERS

# https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

###  JOIN WORKERS

# kubeadm join 192.168.8.11:6443 --token e538xq.2s80p3l2psq07urk --discovery-token-ca-cert-hash sha256:b64f8d49f5c005d3339ddb57cf46896a23db8d40d5875de820250fbd974c5df5

##You can now join any number of machines by running the following on each node as root:

#  kubeadm join 192.168.8.11:6443 --token ec7kwz.w7pn329ssota8lfi --discovery-token-ca-cert-hash sha256:6bb4f27020e9a106ea30c4a1bbc41f84631de8c1d73a96a16fb07c9dbbd6ed45 --server=${KUBE_CONTEXT}




