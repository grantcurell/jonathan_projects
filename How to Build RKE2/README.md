# RKE2 Cluster SOP

**This document provides a comprehensive, step-by-step guide for setting up a small RKE2 (Rancher Kubernetes Engine 2) cluster using VirtualBox and Red Hat Enterprise Linux (RHEL) 8.10. It covers the entire process from creating virtual machines to deploying a simple web application on the cluster. Key steps include:**

- Setting up VirtualBox and creating VMs for the master and worker nodes
- Installing and configuring RHEL 8.10 on the VMs
- Configuring networking for the cluster nodes
- Installing and setting up RKE2 on both master and worker nodes
- Verifying cluster functionality
- Deploying a basic Apache web server (httpd) application to test the cluster

**This guide is ideal for those looking to gain hands-on experience with Kubernetes cluster setup and management in a controlled, virtualized environment. It provides practical insights into the process of creating a functional Kubernetes infrastructure using RKE2 on RHEL systems.**

---

This guide assumes the use of the following tools and software versions:

- [VirtualBox](https://www.virtualbox.org/): Used for creating and managing virtual machines
- [Red Hat Enterprise Linux (RHEL) 8.10](https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/auth?client_id=account-console&redirect_uri=https%3A%2F%2Fsso.redhat.com%2Fauth%2Frealms%2Fredhat-external%2Faccount%2F&state=8b281444-b81f-4e57-8ae0-834cdc56517c&response_mode=query&response_type=code&scope=openid&nonce=bda66b5c-022f-4076-8c16-b1128abd3ebe&code_challenge=_ib18xO611Um7GKLn_TVJw4klrSTcyO8hAmJpAUkeSE&code_challenge_method=S256): The operating system for all cluster nodes
- [PuTTY](https://www.putty.org/): SSH client for connecting to and managing the virtual machines

Ensure you have these tools installed and properly configured on your system before proceeding with the cluster setup. Let’s begin!

<aside>
💡

## Creating VMs Virtual Box

- **Open VirtualBox** and click on **"New"** to create a new VM.
- **Name the VM** (e.g., Master) and select:
    - **Type**: Linux
    - **Version**: Red Hat (64-bit) (or the appropriate version you’re using)
- **Allocate CPUs** — at least 4 for a minimal setup.
- **Allocate Memory** — at least 8gb for a minimal setup.
- **Create a Virtual Hard Disk**:
    - Choose **Create a virtual hard disk now** and click **Create**.
    - Choose **VDI (VirtualBox Disk Image)** and click **Next**.
    - Choose a size (e.g., **20 GB**) and click **Create**.
- **Attach an ISO file** of the Red Hat operating system:
    - Go to **Settings** > **Storage**.
    - Under **Controller: IDE**, click **Empty**, then click the disk icon on the right to select your ISO file.
- Follow the same steps to create another VM (e.g., Worker).
- Allocate similar resources (RAM, disk) and use the same network settings
</aside>

<aside>
💡

## Setting Up Virtual Networking

We need to add a Bridge Adapter so we can allow traffic between both VMs and have both connect to the Internet.

On your PC, open CMD > Ipconfig and find your Default Gateway.

![image.png](images/image.png)

Note the Subnet /24, this will be the network your VMs will be on and we will have to assign them IPs.

Also we need the MAC address of the host’s and put them in replace of the generated MAC addresses inside VirtualBox.

![image.png](images/image%201.png)

In Each VM:

- Go to Settings > Networking
    - Add another network device `Bridge Adapter` , Also replace the MAC address with your host MAC address for both adapters and Save.
        
        ![image.png](images/image%202.png)
        
- Turn on both VMs
</aside>

<aside>
💡

## Installing RHEL 8.10

- **Start each VM** and follow the prompts to install the OS from the ISO.
    - During installation, set up a **hostname** for each VM (e.g., `master` and `worker`)
    - Enable  the Network Adapter enp0s3 and add the following:
        - Switch IPv4 to Manual
        - add IP address,
        - add Default Gateway
        - add DNS Server
    
    | VM | IP Address | Subnet Mask | Default Gateway | DNS Server |
    | --- | --- | --- | --- | --- |
    | Master | 192.168.1.200 | 24 | 192.168.1.1 | 8.8.8.8 |
    | Worker | 192.168.1.201 | 24 | 192.168.1.1 | 8.8.8.8 |
    - Register to RHEL if applicable
    - Create a password for root
    - **Reboot** the VMs after installation.
</aside>

<aside>
💡

## Verify Network Connectivity

For both Master and Worker, check connectivity by pinging the each other and the DNS server.

**Test the Connection**:

```bash
#Use this to find the IP for the Server
ip ad sh
```

```bash
#Test the connectivity.
ping <Worker IP> # <- Replace with the IP of the opposite Server
ping 8.8.8.8 
```

Double check that **Automatically Connect** is enabled on both network devices

![This is the Worker VM](images/image%203.png)

This is the Worker VM

</aside>

<aside>
💡

## SSH Access

We’re going to use Putty to SSH into our VMs. But first we have to enable it in both.

```bash
nano /etc/ssh/sshd_config
```

In Both VMs edit the sshd_config. Remove the 4 hashtags exit and save.

![image.png](images/image%204.png)

Open putty and enter in the IP address of the Master VM and click open.

Great! We are ready to begin the process of installing RKE2!

</aside>

<aside>
💡

## Installing RKE2

## How to Disable Swap

Disabling swap ensures that Kubernetes has full control over memory management and operates as designed. This leads to better performance, more reliable pod scheduling, and predictable resource allocation across your cluster nodes.

To disable swap on a Linux system (including RHEL):

- **Temporarily disable swap** (until the next reboot):
    
    ```bash
    swapoff -a
    ```
    
- **Permanently disable swap** by editing `/etc/fstab`:
    - Open `/etc/fstab` and locate any lines that reference swap partitions.
    
    ```bash
    nano /etc/fstab
    ```
    
    - Comment out those lines by adding a `#` at the beginning of each line.
    - Save and close the file.

## Updating Servers and Disabling Firewall

For Kubernetes we will need to "set" one of the nodes as the control plane and in this guide we will use Master. SSH into both nodes and make sure we have all the updates and add a few things.

```bash
# Rocky instructions 
# stop the software firewall
systemctl disable --now firewalld

# get updates, install nfs, and apply
yum install -y nfs-utils cryptsetup iscsi-initiator-utils

# update all the things
yum update -y

# clean up
yum clean all
```

</aside>

<aside>
💡

## **RKE2 Server Install (RKE2-Master)**

Now that we have all the nodes up to date, let's focus on Master. While this might seem controversial, `curl | bash` does work nicely. The install script will use the tarball install for **Ubuntu** and the RPM install for **Rocky/Centos**. Please be patient, the start command can take a minute. Here are the [rke2 docs](https://docs.rke2.io/install/methods/) and [install options](https://docs.rke2.io/install/configuration#configuring-the-linux-installation-script) for reference.

```bash
# On Master
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh - 

# we can set the token - create config dir/file
mkdir -p /etc/rancher/rke2/ 
echo "token: bootstrapAllTheThings" > /etc/rancher/rke2/config.yaml

# start and enable for restarts - 
systemctl enable --now rke2-server.service
```

Here is what it should look like:

![https://github.com/clemenko/rke_install_blog/raw/main/img/rke_install.jpg](https://github.com/clemenko/rke_install_blog/raw/main/img/rke_install.jpg)

Let's validate everything worked as expected. Run a `systemctl status rke2-server` and make sure it is `active`.

Perfect! Now we can start talking Kubernetes. We need to symlink the `kubectl` cli on Master that gets installed from RKE2.

![https://github.com/clemenko/rke_install_blog/raw/main/img/rke_status.jpg](https://github.com/clemenko/rke_install_blog/raw/main/img/rke_status.jpg)

```bash
# symlink all the things - kubectl
ln -s $(find /var/lib/rancher/rke2/data/ -name kubectl) /usr/local/bin/kubectl

# add kubectl conf with persistence, as per Duane
echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml PATH=$PATH:/usr/local/bin/:/var/lib/rancher/rke2/bin/" >> ~/.bashrc
source ~/.bashrc

# check node status
kubectl get node
```

Hopefully everything looks good! Here is an example.

![https://github.com/clemenko/rke_install_blog/raw/main/img/rke_nodes.jpg](https://github.com/clemenko/rke_install_blog/raw/main/img/rke_nodes.jpg)

For those that are not TOO familiar with k8s, the config file is what `kubectl` uses to authenticate to the api service. If you want to use a workstation, jump box, or any other machine you will want to copy `/etc/rancher/rke2/rke2.yaml`. You will want to modify the file to change the ip address.

</aside>

<aside>
💡

## **RKE2 Agent Install (**Worker**)**

The agent install is honestly the same as server install. Except that we need an agent config file before starting. We need to install the agent and setup the configuration file.

The worker node requires a valid token to join the cluster as a worker. 

```bash
# we can export the rancher-01 IP from the first server.
export RANCHER1_IP=192.168.1.200   # <-- change this!

# we add INSTALL_RKE2_TYPE=agent
curl -sfL [https://get.rke2.io](https://get.rke2.io/) | INSTALL_RKE2_TYPE=agent sh -

# create config dir/file
mkdir -p /etc/rancher/rke2/

# change the ip to reflect your rancher-01 ip
cat << EOF >> /etc/rancher/rke2/config.yaml
server: https://$RANCHER1_IP:9345
token: bootstrapAllTheThings
EOF

# enable and start
systemctl enable --now rke2-agent.service
```

Now we can run these set of commands.

What should this look like:

![https://github.com/clemenko/rke_install_blog/raw/main/img/rke_agent.jpg](https://github.com/clemenko/rke_install_blog/raw/main/img/rke_agent.jpg)

Rinse and repeat. Run the same install commands on any other nodes you might have. Next we can validate all the nodes are playing nice by running.

*If the worker node is missing a role, you can add the role manually. Run the following command on the master node:*

![https://github.com/clemenko/rke_install_blog/raw/main/img/moar_nodes.jpg](https://github.com/clemenko/rke_install_blog/raw/main/img/moar_nodes.jpg)

```bash
kubectl label node worker node-role.kubernetes.io/worker=true
```

`kubectl get node` on Master.

</aside>

<aside>
💡

## Deploying the httpd Container

In Kubernetes, we’ll use a ConfigMap to store the custom HTML content that the `httpd` container will serve. We will create 3 files that can be saved in a new directory.

### **Create a Project Directory**

You can store the YAML files anywhere on your system, but it’s common practice to keep them organized in a dedicated directory for easy access and management. Here are some recommendations:

- For example, create a directory specifically for your RKE2 cluster setup:
    
    ```bash
    mkdir -p ~/rke2-project/deployments
    cd ~/rke2-project/deployments
    ```
    

### 2. **Store YAML Files in This Directory**

- Save your YAML files here. For example:
    - `httpd-configmap.yaml` for the ConfigMap
    - `httpd-deployment.yaml` for the Deployment
    - `httpd-service.yaml` for the Service
- This organization makes it easy to keep all related files together and track changes as you work on the project.

### Create a ConfigMap for Custom HTML Content

We'll use a ConfigMap to store the custom HTML content for the `httpd` container.

1. **Create a YAML file** named `httpd-configmap.yaml` with the following content:
    
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: httpd-html
    data:
      index.html: |
        <html>
          <body>
            <h1>We did it Reddit</h1>
          </body>
        </html>
    
    ```
    
2. **Apply the ConfigMap** to the cluster:
    
    ```bash
    kubectl apply -f httpd-configmap.yaml
    ```
    

This ConfigMap will store a file called `index.html` with the custom message “We did it Reddit.”

### Step 2: Create a Deployment for the `httpd` Pod

Next, we’ll create a Deployment that uses the `httpd` container and mounts the ConfigMap with the custom HTML as the default webpage.

1. **Create a YAML file** named `httpd-deployment.yaml` with the following content:
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: httpd-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: httpd
      template:
        metadata:
          labels:
            app: httpd
        spec:
          containers:
          - name: httpd
            image: httpd:latest
            ports:
            - containerPort: 80
            volumeMounts:
            - name: html
              mountPath: /usr/local/apache2/htdocs/
          volumes:
          - name: html
            configMap:
              name: httpd-html
    ```
    
    - **Explanation**:
        - The `httpd` container uses the official `httpd` image.
        - The `volumeMounts` section mounts the `html` volume (linked to the `httpd-html` ConfigMap) at `/usr/local/apache2/htdocs/`, which is the default directory for `httpd` to serve files.
2. **Apply the Deployment**:
    
    ```bash
    kubectl apply -f httpd-deployment.yaml
    ```
    

### Step 3: Expose the `httpd` Deployment Using a NodePort Service

To access the `httpd` container from outside the cluster, create a NodePort service.

1. **Create a YAML file** named `httpd-service.yaml` with the following content:
    
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: httpd-service
    spec:
      selector:
        app: httpd
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
          nodePort: 30080  # Specify a port in the NodePort range (30000-32767)
      type: NodePort
    ```
    
    - **Explanation**:
        - The service listens on port `80` and forwards traffic to the `httpd` pod on the same port.
        - `nodePort: 30080` exposes the service on port `30080` on each node in the cluster, allowing external access.
2. **Apply the Service**:
    
    ```bash
    kubectl apply -f httpd-service.yaml
    ```
    

### Step 4: Access the `httpd` Application

1. **Get the IP address of a node in your cluster**:
    
    ```bash
    kubectl get nodes -o wide
    ```
    
2. **Access the `httpd` webpage**:
    - Use a browser or `curl` to visit `http://<Node_IP>:30080`.
    - You should see the custom webpage displaying **"We did it Reddit"**.
</aside>

<aside>
💡

### Access the httpd Application

1. **Access the httpd webpage**:
    - Open a browser or use `curl` to access `http://<Node_IP>:30080`.
    - You should see the message: **"We did it Reddit"**.

This setup deploys the `httpd` container with your custom HTML content on the RKE2 cluster and exposes it to external traffic. 

</aside>

---

## References

https://github.com/clemenko/rke_install_blog?tab=readme-ov-file

[https://github.com/mevijays/training-k8s/blob/main/docs/rke2.md](https://github.com/mevijays/training-k8s/blob/main/docs/rke2.md)