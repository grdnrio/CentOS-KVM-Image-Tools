# CentOS KVM Image Tools

Some simple tools, instructions and images to assist with creating CentOS KVM virtual machines. The following guide assumes you have virtualisation tools such as virt-install and libguestfs tools such as virt-sparsify installed.

## Testing virt-install

### Simple CentOS 6.x install from a remote HTTP kickstart file, using GUID partition tables

This command will perform a text-based installation of the latest CentOS release (currently CentOS 6.3) directly from a public HTTP mirror without requiring any installation CD/DVD. A virtual machine will be created with the given settings, and output from the installation will be sent to your terminal rather than a VNC session.

    virt-install \
    --name "centos6x-vm-gpt" \
    --ram 1024 \
    --nographics \
    --os-type=linux \
    --os-variant=rhel6 \
    --location=http://mirror.catn.com/pub/centos/6/os/x86_64 \
    --extra-args="ks=http://fubralimited.github.com/CentOS-KVM-Image-Tools/kickstarts/centos6x-vm-gpt.cfg text console=tty0 utf8 console=ttyS0,115200" \
    --disk path=/var/lib/libvirt/images/centos6x-vm-gpt.img,size=10,bus=virtio,format=qcow2
    
Once the installation is complete, you can connect to the virtual machine's console with:

    virsh console centos6x-vm-gpt
    
If you want to delete the virtual machine, you can do so with:

    virsh destroy centos6x-vm-gpt
    virsh undefine centos6x-vm-gpt
    rm /var/lib/libvirt/images/centos6x-vm-gpt.img

## CentOS 6.x Gold Master Image

### Creating a CentOS Golden Master Image    

1) Install a fresh virtual machine that will become our base image

    virt-install \
    --name "centos6.3-gold" \
    --ram 1024 \
    --nographics \
    --os-type=linux \
    --os-variant=rhel6 \
    --location=http://mirror.catn.com/pub/centos/6/os/x86_64 \
    --extra-args="ks=http://fubralimited.github.com/CentOS-KVM-Image-Tools/kickstarts/centos6x-vm-gpt.cfg text console=tty0 utf8 console=ttyS0,115200" \
    --disk path=/var/lib/libvirt/images/centos6.3-gold.img,size=10,bus=virtio,format=qcow2

2) Make sure you are inside the new guest, then apply the latest package updates from yum…
    
    yum update
    reboot
    
3) The guest should have rebooted, once it comes back up - log in, and then remove any old kernels to free up some space (assuming the kernel was updated in the previous step

    yum remove $(rpm -q kernel | grep -v `uname -r`)
    
4) Run the create gold master bash script to remove MAC address references etc.. then shut it down.

Note: If you are using a newer version of libguestfs-tools then you could try using virt-sysprep -a imagename instead of the create-gold-master.sh script. It seems there is a bug with the earlier versions, like the one shipped with Ubuntu precise, whereby it wouldn't detect rhel OS types, and therefore doesn't run the parts to wipe the hostname or remote the mac addresses references - https://bugzilla.redhat.com/show_bug.cgi?id=811112.

So if you are using Ubuntu Precise as your hypervisor, run the following commands from within the guest.

    wget https://raw.github.com/fubralimited/CentOS-KVM-Image-Tools/master/scripts/create-gold-master.sh;
    bash create-gold-master.sh
    shutdown -h now
    
Whereas if you are running Centos 6.3 as the hypervisor (with the newer version of libguestfs-tools), you can run

    shutdown -h now
    virt-sysprep -a /var/lib/libvirt/images/centos6.3-gold.img
    
    
5) From the hypervisor sparsify and compress the VM image

    cd /var/lib/libvirt/images/;
    virt-sparsify centos6.3-gold.img centos6.3-gold.img-sparsified
    qemu-img convert -c -p -f qcow2 -O qcow2 centos6.3-gold.img-sparsified centos6.3-gold-master.img
    
### Creating a new guest using a copy of the Golden Master image

Copy the image

    cp centos6.3-gold-master.img centos6.3-gold-copy1-nobacking.img
    
Create a new guest using this image with virt-install --import

    virt-install \
    --name centos6.3-gold-copy1-nobacking \
    --ram 1024 \
    --os-type=linux \
    --os-variant=rhel6 \
    --disk path=/var/lib/libvirt/images/centos6.3-gold-copy1-nobacking.img \
    --import

### Creating a new guest using the Golden Master as a backing image

Create a new image using qemu-img that specifies the master as the backing image

    qemu-img create -f qcow2 -b /var/lib/libvirt/images/centos6.3-gold-master.img /var/lib/libvirt/images/centos6.3-gold-copy2-master-backed.img

Create a new guest using this image with virt-install --import

    virt-install \
    --name centos6.3-gold-copy2-master-backed \
    --ram 1024 \
    --os-type=linux \
    --os-variant=rhel6 \
    --disk path=/var/lib/libvirt/images/centos6.3-gold-copy2-master-backed.img \
    --import

## Working with images

### Resizing a virtual machine image

Coming soon...