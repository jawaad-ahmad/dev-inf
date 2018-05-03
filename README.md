dev-inf
================================================================================

Infrastructure as code for development projects


Introduction
================================================================================

This document sets up an initial VM environment to bootstrap the Ansible push
server.

Important: The entire point of this is to be able to set up a basic starting
point VM without the Ansible environment until Ansible can take over; therefore,
all steps must be performed manually.


Reference
================================================================================

  * https://medium.com/@mvuksano/automated-provisioning-of-virtual-machines-for-development-8a543e435f44


Variables
================================================================================

The following variables are used throughout this procedure:

   | Key              | Value        |
   |------------------|--------------|
   | INSTALL_ISO_PATH | /path/to/downloaded/Debian/jessie/debian-8.5.0-amd64-DVD-1.iso |
   | EXPORTED_VMS     | /path/to/ova |
   | VMNAME           | vdi          |
   | HOSTNAME         | vdi          |
   | USER_FULLNAME    | Ansible      |
   | USERNAME         | ansible      |
   | PASSWORD         | ansible      |


Bootstrap Base Install
================================================================================

**Note:** The following expects a network connection is available to the
Internet.

Download Debian ISO (TODO which one?).

    $ git clone https://github.com/jawaad-ahmad/dev-inf


**Note:** The remainder of this section assumes no network connection is
available. If the steps above have been completed adequately, then the
remainder of this procedure can be completed offline.

    $ cd dev-inf
    $ ./init.sh
    $ common-inf/scripts/create-vbox-vm.sh $(VMNAME) auto 512 10240 ${INSTALL_ISO_PATH}

(**TODO:** Does create-vbox-vm.sh create the VM with host-only networking? The steps below appear to assume host-only on the 192.168.56.x subnet for eth1.)

Start the VM. Follow the prompts:

  * Boot menu: Install
  * Select a language: English
  * Select your location: United States
  * Configure the keyboard: American English
  * Configure the network:
     * eth1
     * Continue without a default route? Yes [TODO]
     * Name server addresses: (blank) [TODO]
     * Hostname: $(HOSTNAME)
     * Domain name: (blank)
  * Set up users and passwords:
     * Root password: (blank)
     * Re-enter root password: (blank)
     * Full name: $(USER_FULLNAME)
     * Username: $(USERNAME)
     * Password: $(PASSWORD)
     * Re-enter password: $(PASSWORD)
  * Configure the clock: Central [TODO]
  * Partition disks
     * Guided - use entire disk
     * Select disk: SCSI1
     * Partitioning scheme: All files in one partition
     * Finish partitioning and write changes to disk
     * Write changes to disk? Yes
  * Configure the package manager
     * Scan another CD or DVD? No
     * Use a network mirror? No
  * Configure popularity-content: No
  * Software selection:
     * Uncheck Debian desktop environment
     * Uncheck print server
     * Check SSH server
     * Leave standard system utilities checked
     * Continue
  * Install the GRUB boot loader on a hard disk:
     * Yes
     * /dev/sda
  * Finish the installation: Continue

Remove an installation CDs or USB drives and reboot the VM.

Set BIOS boot order as applicable to boot from the appropriate drive.


Verify Networking
================================================================================

Log-in as $(USERNAME).

    vdi$ /sbin/ifconfig -a

Verify eth1 has an IP address available: 192.168.56.___

From the physical host, try to ssh into the VM using this IP address.

    host$ vdi_ip_addr=192.168.56.___
    host$ ssh ansible@${vdi_ip_addr}
    Enter password:
    vdi$

From the VM, verify the host is available:

    vdi$ ssh user@192.168.56.1
    Enter password:
    host$ exit
    vdi$ exit
    host$


Install SSH Authorized Key
================================================================================

    host$ cat ~/.ssh/id_rsa.pub | ssh ansible@${vdi_ip_addr} 'cat - >> ~/.ssh/authorized_keys'
    Enter password:
    host$ ssh ansible@${vdi_ip_addr}
    vdi$

Verify logged-in with no password prompt.


Install Packages
================================================================================

(TODO Now this assumes we have a connection to the Internet. When did this happen?)

    vdi$ sudo -i
    root# mv /etc/apt/sources.list /etc/apt/sources.list.orig
    root# sed -e 's/^\(deb\ cdrom\:\)/\#\1/' < /etc/apt/sources.list.orig > /etc/apt/sources.list
    root# cat << END_OF_FILE > /etc/apt/sources.list.d/tempsrc.list
    deb ftp://ftp.us.debian.org/debian jessie main
    deb ftp://ftp.us.debian.org/debian jessie non-free
    deb ftp://ftp.us.debian.org/debian jessie contrib
    END_OF_FILE
    root# apt-get update
    root# apt-get install --yes cloud-init
    root# /bin/rm /etc/apt/sources.list.d/tempsrc.list
    root# apt-get update
    root# echo TODO - clean-up - prepare for export (zero-wipe free space, remove keys, etc???)
    root# poweroff


Export Base VM
================================================================================

    host$ VBoxManage export ${VMNAME} --output ${EXPORTED_VMS}/vdi-base.ova


Create cloud-init Disc
================================================================================

(TODO Can this be changed to use an offline copy of Ansible instead of the PPA?)

(TODO Change to use the dev-inf repo)

    host$ CI_HOME=/tmp/cloud-init
    host$ cat ~/.ssh/id_rsa.pub
    ssh-rsa XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

Copy the contents of the public key. Paste below when entering the contents of
user-data.

    host$ mkdir ${CI_HOME}
    host$ cat << END_OF_FILE > ${CI_HOME}/user-data
    #cloud-config
    write_files:
      - path: /etc/apt/sources.list.d/cloud-init.tmp.list
        content: |
          deb ftp://ftp.us.debian.org/debian jessie main contrib non-free
        owner: root:root
        permissions: 644
    
    #manage_etc_hosts: true
    apt_sources:
      - source: "ppa:ansible/ansible"
    
    packages:
      - ansible
      - git
    
    runcmd:
      - /bin/rm /etc/apt/sources.list.d/cloud-init.tmp.list
      - 'git clone https://gitlab.com/markovuksanovic/playbooks.git'
      - 'ansible-playbook -vvvv /playbooks/dev-machine.yml >>/tmp/ansible-output.txt 2>&1'
    END_OF_FILE
    host$ cat << END_OF_FILE > ${CI_HOME}/meta-data
    hostname: vdi
    local-hostname: vdi
    END_OF_FILE
    host$ genisoimage -output ${CI_HOME}/seed.iso -volid cidata -R -J ${CI_HOME}/user-data ${CI_HOME}/meta-data


Provision Base Instance
================================================================================

    host$ VBoxManage import ${EXPORTED_VMS}/vdi-base.ova --vsys 0 --vmname vdi2
    host$ VBoxManage storageattach vdi2 --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium ${CI_HOME}/seed.iso

Start the VM:

(TODO: --type headless??? No - can't troubleshoot problems)

    host$ VBoxManage startvm vdi2

Monitor progress (TODO if headless):

    host$ while [ 1 ]; do VBoxManage showvminfo vdi2 | grep State; sleep 1; done

Wait for state to go from running to powered off.


TODO...




$ genisoimage -output ${CI_HOME}/seed.iso -volid cidata -R -J ${CI_HOME}/user-data ${CI_HOME}/meta-data

$ while [ 60 ]; do VBoxManage showvminfo vdi2 | grep State; sleep 1; done

$ VBoxManage controlvm vdi2 poweroff && VBoxManage unregistervm vdi2 --delete; VBoxManage import "${EXPORTED_VMS}/vdi-base.ova" --vsys 0 --vmname vdi2 && VBoxManage storageattach vdi2 --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium ${CI_HOME}/seed.iso && VBoxManage startvm vdi2


Cleanly power off the VM. (I
generally do this by allowing the VM to reboot, then pressing the hotkey [e.g.
F12] to get into the boot options right at startup, and then powering off the
VM hard from the Virtual Box Manager.)

