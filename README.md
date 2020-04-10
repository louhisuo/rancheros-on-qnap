# RancherOS on QNAP
Long story short ... in search of docker and kubernetes homelab environments which works on QNAP NAS and requres as minimal maintance as possible allowing me focus working with docker/k8s I have been going through different alternaternatives e.g. Ubuntu, CentOS to name few. Props has been that these are full blown Linux distros with all bells and wishles but cons is that I need to spend quite some time to provision a node (or host or VM whichever you may want to call it) before a node is actually usable. This probably improves in future after I learn how to use cloud-init but this is where I stand at the moment.

So I decided to look into Container Operating systems e.g. CoreOS (Red Hat), PhotonOS (VMware), RancherOS (Rancher Labs), to name few, and decided to give RancherOS try as it looked fairy straight forward to setup. So far outcome is so promising that I decided to create a hort guide to document how to spin up quickly docker nodes with RancherOS, without steps of VM cloning etc.

## Pre-installation steps
These steps are only needed to be executed once (as longs as you store/document outcome).

Download an RancherOS image from (I am currently using rancheros.iso)

    https://github.com/rancher/os/blob/master/README.md

Generate SSH keypair for a remote SSH access from your client machine into RancherOS VM and note your public key (you need it for cloud-config.yaml). In your SSH client machine execute following steps.

    client$ ssh-keygen
    client$ cat $HOME/.ssh/id_rsa.pub

Generate Docker Hub access token to able to push dokcer images to Hub

    - In your Docker Hub profile view click 'Edit profile'-> 'Security'
    - Click 'New Access Token' to create a new access token.
    - Note a token for later use (you need it for cloud-config.yaml)

## Create cloud-config.yaml
This is an example 'cloud-config.yaml' (created in my client machine9. You need to have a unique yaml-file for each node you create. 

    # cloud-config
    hostname: docker-ce
    ssh_authorized_keys:
      - <place your id_rsa.pub content here>
    # Optional, in case you need access external filesystems inside of your RancherOS node.
    mounts:
      - ["<nfs-server-ip>:/rancheros", "/storage", "nfs4", ""]
    rancher:
      # enable VM tools and NFS client functionality
      services_include:
        qemu-guest-agent: true
        volume-nfs: true
      network:
        interfaces:
          eth0:
            dhcp: false
            address: 192.168.51.10/24
            gateway: 192.168.51.254
        dns:
          nameservers:
            - 1.1.1.1
            - 1.0.0.1
          override: true
        # auth is base64 from your docker hub user id and access token you created earlier using format 'username:token'
        registry_auths:
          https://index.docker.io/v1/:
            auth: <place base64 from 'username:token' here>

# Create a new persistant RancherOS node VM
Create VM by using QNAP Virtualization Station. These steps are needed for every node you create with an unique 'cloud-config.yaml' you have created in above step from above example.In 'Create VM' window I use following settings.   
Note: Boot from ISO image (earlier downloaded rancheros.iso) requires minimal 2048MB to boot up.

    - OS type: Generic
    - Memory: 2048 MB (note: with 1024 MB) VM will not boot up)
    - CD image: rancheros.iso
    - HDD storage: 8 GB
    - Network -> Connect to: <choose one of networks you have available in drop down menu>

After RancherOS VM is up and running, connect it with Virtualization Station console and print IP address of VM

    $ ip address

Set 'rancher' user password. RancherOS normally does not use password based authentication and strongly recommends to use SSH key based login but this step is needed to transfer 'cloud-config.yaml' into VM.

    $ sudo password rancher

Transfer 'cloud-config.yaml into VM from your client machine.

    client$ sftp rancher@<vm-ip-address>
    > put cloud-config.yaml
    > exit

Login with SSH into RancherOS VM

    client$ ssh -l rancher <vm-ip-address>

Install RancherOS to a disk (for persistant installation) in order of RancherOS to able to store its state to dick.

    $ sudo ros install -c cloud-config.yaml -d /dev/sda
    Note: Press 'y' to confirm installation step

DO NOT REBOOT when you get a promt "Continue with reboot [y/N]:' instead execute following steps from Virtualization station

    - Shutdoewn VM
    - Remove mount to 'rancheros.iso'from VM (click 'Eject CD image at CD/DVD configuration section)
    - Start VM

After VM was started, test your installation

    # login into VM with SSH
    client$ ssh -l rancher <vm-ip-address>
    # Check that docker is running
    $ docker version
    # Check that NFS mount was successful (you should see content of 'rancheros' directory within external system)
    $ ls -al /storage
    
    - In Virtualization Station check that IP address of VM is visible under 'Network' section when you browse VM 'Information' section

Note about console access:  
As default you will not be able to login via console into RancherOS vM. If you need console access into a RancherOS node VM, on VM console screen select 'Autologin on tty1 amd ttyS0' bootloader option to enable 'rancher' user autologin.

References:
---
- https://rancher.com/rancher-os/  
- https://rancher.com/docs/os/v1.x/en/overview/  
- https://github.com/rancher/os/blob/master/README.md  
- https://linuxhint.com/install_rancher_os/  
- https://distrowatch.com/table.php?distribution=rancheros  
- https://vitobotta.com/2019/06/25/setting-up-rancheros-for-rancher-and-kubernetes/  

TO-DO:
----
- Docker Hub login with token does not work, will be fixed later stage. As workaround login manually with 'docker login'
- How to deploy 'rancher-compose', equalent to 'docker-compose' on Rancher ecosystem up and running
- Add instructions how to create multiple network interfaces setup (requires defining another "physical" adapter/interface for VM via Virtualization Station 
