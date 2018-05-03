dev-inf
================================================================================

Infrastructure as code for development projects


Introduction
================================================================================

This document sets up an initial VM environment to bootstrap a local
development VDI.

Important: The entire point of this is to be able to set up a basic starting
point VM from scrach without the Ansible environment until Ansible can take
over; therefore, all steps must be performed manually.

Running through this document will provide you with a VM OVA file and a
cloud-init ISO file that you can you repeatedly to quickly generate a
development VDI as often as desired from a base configuration maintained in a
git repository.


References
================================================================================

  * https://medium.com/@mvuksano/automated-provisioning-of-virtual-machines-for-development-8a543e435f44
  * https://lonesysadmin.net/2013/03/26/preparing-linux-template-vms/


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


Create cloud-init Disc
================================================================================

**Note:** This section only needs to be run once to create the seed.iso
file. If this section was previously run, then it can be skipped unless
rebuild of the seed.iso is desired.

**Note:** This section can be postponed if desired. The following section(s)
will take additional time to complete. If desired, skip to the next section to
get the VM installation going, and then come back to run through these steps
while waiting for the installation to complete.

(TODO Can this be changed to use an offline copy of Ansible instead of the PPA?)

(TODO Change to use the dev-inf repo)

In a terminal, run the following:

    $ CI_HOME=/tmp/cloud-init
    $ mkdir -p "${CI_HOME}"

Create the user-data file:

    $ cat << END_OF_FILE > "${CI_HOME}/user-data"
    #cloud-config
    write_files:
      - path: /etc/apt/sources.list.d/cloud-init.tmp.list
        content: |
          deb ftp://ftp.us.debian.org/debian jessie main contrib non-free
        owner: root:root
        permissions: 644
    
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

Create the meta-data file:

    $ cat << END_OF_FILE > "${CI_HOME}/meta-data"
    hostname: vdi
    local-hostname: vdi
    END_OF_FILE

Generate the cloud-init seed ISO:

    $ mkdir -p "${EXPORTED_VMS}/cloud-init"
    $ genisoimage -output "${EXPORTED_VMS}/cloud-init-seed.iso" -volid cidata -R -J "${CI_HOME}/user-data" "${CI_HOME}/meta-data"


Bootstrap Base Install
================================================================================

**Note:** This section only needs to be run once to create the vdi-base.ova
file. If this section was previously run, then it can be skipped unless
rebuild of the vdi-base.ova is desired.

**Note:** The following expects a network connection is available to the
Internet.

1. Download Debian ISO (TODO which one?).

1. Clone the git repo containing the VDI infrastructure.

    $ cd ~/repo
    $ git clone https://github.com/jawaad-ahmad/dev-inf
    $ cd dev-inf
    $ ./init.sh

(TODO init.sh does not yet exist.)

**Note:** The remainder of this section assumes no network connection is
available. If the steps above have been completed adequately, then the
remainder of this procedure can be completed offline.

    $ common-inf/scripts/create-vbox-vm.sh ${VMNAME} auto 512 10240 ${INSTALL_ISO_PATH}

(**TODO:** Does create-vbox-vm.sh create the VM with host-only networking? (No.) The steps below appear to assume host-only on the 192.168.56.x subnet for eth1.)

1. Start the VM:

    $ VBoxManage startvm ${VMNAME}

Follow the prompts:

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
  * Configure popularity-contest: No
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

Remove any installation CDs or USB drives and reboot the VM. Do not power off
the VM at this point; let it continue to come up.

Set BIOS boot order as applicable to boot from the appropriate drive.


Verify Networking
================================================================================

**Note:** This section only needs to be run once to create the vdi-base.ova
file. If this section was previously run, then it can be skipped unless
rebuild of the vdi-base.ova is desired.

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


Install Cloud-Init
================================================================================

**Note:** This section only needs to be run once to create the vdi-base.ova
file. If this section was previously run, then it can be skipped unless
rebuild of the vdi-base.ova is desired.

(TODO Now this assumes we have a connection to the Internet. When did this happen?)

Run the following in a terminal on the VDI host:

    vdi$ sudo -i
    root# sed --in-place=.orig -e 's/^\(deb\ cdrom\:\)/\#\1/' < /etc/apt/sources.list.orig > /etc/apt/sources.list
    root# echo deb ftp://ftp.us.debian.org/debian jessie main non-free contrib > /etc/apt/sources.list.d/tempsrc.list
    root# apt-get update
    root# apt-get install --yes cloud-init


Export Base VM
================================================================================

**Note:** This section only needs to be run once to create the vdi-base.ova
file. If this section was previously run, then it can be skipped unless
rebuild of the vdi-base.ova is desired.

Run the following in a terminal on the VDI host to clean up the VM hard drive
and power off the VM:

    vdi$ sudo -i
    root# /bin/rm /etc/apt/sources.list.d/tempsrc.list

TODO... https://lonesysadmin.net/2013/03/26/preparing-linux-template-vms/

    root# echo TODO - clean-up - prepare for export (zero-wipe free space, remove keys, etc???)
    root# poweroff

On the host, run the following to create the base VM OVA:

    host$ VBoxManage export ${VMNAME} --output "${EXPORTED_VMS}/vdi-base.ova"

Remove the VM from the VirtualBox inventory. This will delete all files
pertaining to the VM, but it's OK since we can get them back from vdi-base.ova.

    host$ VBoxManage unregistervm ${VMNAME} --delete


Provision Base Instance
================================================================================

Using the vdi-base.ova and the cloud-init-seed.iso built above, this section
provisions a new VDI.

    host$ VBoxManage import "${EXPORTED_VMS}/vdi-base.ova" --vsys 0 --vmname ${VMNAME} && VBoxManage storageattach ${VMNAME} --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "${EXPORTED_VMS}/cloud-init-seed.iso"

Start the VM:

    host$ VBoxManage startvm ${VMNAME}

Wait for the VM to power itself down. This should take around TODO minutes.

If desired, tail /var/log/syslog on the VM to monitor cloud-init progress.

If there is any trouble, run the following to destroy and recreate the VM:

    host$ VBoxManage controlvm ${VMNAME} poweroff; VBoxManage unregistervm ${VMNAME} --delete; sleep 2; VBoxManage import "${EXPORTED_VMS}/vdi-base.ova" --vsys 0 --vmname ${VMNAME} && VBoxManage storageattach ${VMNAME} --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "${EXPORTED_VMS}/cloud-init-seed.iso" && VBoxManage startvm ${VMNAME}

Note that the command above will power down the VM hard.


TODO Delete....
=================

Monitor progress (TODO if headless):

    host$ while [ 1 ]; do VBoxManage showvminfo ${VMNAME} | grep State; sleep 30; done

Wait for state to go from running to powered off.



Cleanly power off the VM. (I
generally do this by allowing the VM to reboot, then pressing the hotkey [e.g.
F12] to get into the boot options right at startup, and then powering off the
VM hard from the Virtual Box Manager.)


    host$ CI_HOME=/tmp/cloud-init
    host$ cat ~/.ssh/id_rsa.pub
    ssh-rsa XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

Copy the contents of the public key. Paste below when entering the contents of
user-data.


Install SSH Authorized Key
================================================================================

    host$ cat ~/.ssh/id_rsa.pub | ssh ansible@${vdi_ip_addr} 'cat - >> ~/.ssh/authorized_keys'
    Enter password:
    host$ ssh ansible@${vdi_ip_addr}
    vdi$

Verify logged-in with no password prompt.
