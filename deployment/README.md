Deployment of Kubernetes on OpenStack
======================================


Prepare OpenStack environment
-----------------------------


### Install OpenStack client

```
$ sudo dnf install -y python-openstackclient
```

```
$ sudo pip install python-openstackclient
```


### Create `clouds.yaml` file

```
$ vi ~/.config/openstack/clouds.yaml
```

```
clouds:
  dreamhost:
    auth:
      auth_url: https://iad2.dream.io:5000/v2.0
      project_name: [projectname]
      username: [username]
      password: [password]
    region_name: RegionOne
```

Note: change the values of `[projectname]`, `[username]` and `[password]` accordingly. The values can be found in `stackrc` file you normally use.


```
$ openstack --os-cloud dreamhost server list
```


### Download CentOS Atomic image

```
$ wget -q http://cloud.centos.org/centos/7/atomic/images/CentOS-Atomic-Host-7-GenericCloud.qcow2.gz -O CentOS7-Atomic.qcow2.gz
$ gunzip CentOS7-Atomic.qcow2.gz
```


### Convert image

```
$ dnf install -y qemu-tools
```


```
$ qemu-img convert /tmp/centos7.qcow2 CentOS7-Atomic.raw
```


### Upload image

```
$ openstack --os-cloud dreamhost image create "CentOS7-Atomic" --disk-format raw --container-format bare --file CentOS7-Atomic.raw --property os_distro=centos
```

### Install ansible

```
$ sudo dnf install -y ansible
```

```
$ sudo pip install ansible
```


### Install shade library

```
$ sudo pip install shade
```


### Upload public key

```
$ openstack --os-cloud dreamhost keypair create --public-key ~/.ssh/id_rsa.pub atomic
```


#### Ansible playbook to upload public key

You can also use the following Ansible playbook

```
---
- hosts: localhost

  tasks:
  - name: Upload public key to OpenStack cloud provider
    os_keypair:
      cloud: "{{ cloud }}"
      name: {{ key }}
      public_key_file: ~/.ssh/id_rsa.pub
```

```
$ ansible-playbook upload-publickey.yml -e cloud=dreamhost -e key=atomic
```


### Create instances

```
$ openstack --os-cloud server create --flavor 10 --image "CentOS7-Atomic" --key-name atomic atomic-01
$ openstack --os-cloud server create --flavor 10 --image "CentOS7-Atomic" --key-name atomic atomic-02
$ openstack --os-cloud server create --flavor 10 --image "CentOS7-Atomic" --key-name atomic atomic-03
```

#### Ansible playbook to create instances

```
---
- hosts: localhost

  tasks:
  - name: Create instance
    os_server:
      state: present
      cloud: "{{ cloud }}"
      name: "{{ item }}"
      image: {{ image }}
      key_name: "{{ key }}"
      network: public
      flavor: 10
    with_items:
    - atomic-01
    - atomic-02
    - atomic-03
```

```
$ ansible-playbook create-instance-multiple.yml -e cloud=dreamhost -e key=atomic -e image=CentOS7-Atomic
```


Deploy Kubernetes
-----------------

### Clone playbook repository

```
$ git clone https://github.com/gbraad/ansible-playbook-kubernetes.git
```


### Deploy kubernetes

```
$ openstack --os-cloud dreamhost server list
```


```
$ dnf install -y ansible
$ ansible-galaxy install -r roles.txt
$ vi hosts
$ ansible-playbook -i hosts deploy-kubernetes.yml
```


### Install kubernetes client

```
$ dnf install -y kubernetes-client
```

```
$ wget http://storage.googleapis.com/kubernetes-release/release/v1.3.4/bin/linux/amd64/kubectl
$ chmod +x kubectl
```


### Test

```
$ kubectl --server=[master ip]]:8080 get nodes
```

```
$ kubectl --server=[master ip]]:8080 run nginx
```

```
$ kubectl --server=[master ip]]:8080 get pods
```
