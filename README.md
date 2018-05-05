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

   | Key               | Value          |
   |-------------------|----------------|
   | INSTALL_ISO_PATH  | /path/to/downloaded/Debian/jessie/debian-8.5.0-amd64-DVD-1.iso |
   | EXPORTED_VMS      | /path/to/ova   |
   | ANSIBLE_DATA_HOME | /path/to/files |
   | VM_NAME           | vdi            |
   | HOST_NAME         | vdi            |
   | USER_FULLNAME     | Ansible        |
   | USERNAME          | ansible        |
   | PASSWORD          | ansible        |


Provision Base Instance
================================================================================

Using the vdi-base.ova and the cloud-init-seed.iso built as described below,
this section provisions a new VDI.

    host$ VBoxManage import "${EXPORTED_VMS}/vdi-base.ova" --vsys 0 --vmname ${VM_NAME} && VBoxManage storageattach ${VM_NAME} --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "${EXPORTED_VMS}/cloud-init-seed.iso"

Start the VM:

    host$ VBoxManage startvm ${VM_NAME}

Wait for the VM to power itself down. This should take around TODO minutes.

If desired, tail /var/log/syslog on the VM to monitor cloud-init progress.

If there is any trouble, run the following to destroy and recreate the VM:

    host$ VBoxManage controlvm ${VM_NAME} poweroff; VBoxManage unregistervm ${VM_NAME} --delete; sleep 2; VBoxManage import "${EXPORTED_VMS}/vdi-base.ova" --vsys 0 --vmname ${VM_NAME} && VBoxManage storageattach ${VM_NAME} --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "${EXPORTED_VMS}/cloud-init-seed.iso" && VBoxManage startvm ${VM_NAME}

Note that the command above will power down the VM hard.


Next Steps
================================================================================

Depending on the length of time provisioning took, you might be tempted to
run through the Export Base VM section to make another export of the
provisioned VM. There is nothing wrong with this step, but simply remember that
the exported VM will be frozen in time while the git repository will continue
to evolve.

As a new VM is provisioned off of the OVA, be sure to re-run Ansible upon
provisioning to make sure you have the latest changes.


Internet Downloads
================================================================================

**Note:** This section only needs to be run once to create the seed.iso
file. If this section was previously run, then it can be skipped unless
rebuild of the seed.iso is desired.

**Note:** The following expects a network connection is available to the
Internet.

1. Download Debian ISO (TODO which one?) and save as ${INSTALL_ISO_PATH}.

1. Clone the git repo containing the VDI infrastructure.

    $ cd ~/repo
    $ git clone https://github.com/jawaad-ahmad/dev-inf
    $ cd dev-inf
    $ ./init.sh

(TODO init.sh does not yet exist.)


Create cloud-init-seed.iso
================================================================================

**Note:** This section only needs to be run once to create the seed.iso
file. If this section was previously run, then it can be skipped unless
rebuild of the seed.iso is desired.

**Note:** This section can be postponed if desired. The following section(s)
will take additional time to complete. If desired, skip to the next section to
get the VM installation going, and then come back to run through these steps
while waiting for the installation to complete.

In a terminal on the host, run the following:

    $ CI_HOME=/tmp/cloud-init
    $ mkdir -p "${CI_HOME}"

Copy the cloud-init files:

    $ cp ~/repo/dev-inf/cloud-init/user-data "${CI_HOME}/user-data"
    $ cp ~/repo/dev-inf/cloud-init/meta-data "${CI_HOME}/meta-data"
    $ mkdir -p ${CI_HOME}/software/applications
    $ cp -a {${ANSIBLE_DATA_HOME},${CI_HOME}}/software/applications/ansible

Generate the cloud-init seed ISO:

    $ mkdir -p "${EXPORTED_VMS}/cloud-init"
    $ genisoimage -output "${EXPORTED_VMS}/cloud-init-seed.iso" -volid cidata -R -J "${CI_HOME}"


Bootstrap Base Install
================================================================================

**Note:** This section only needs to be run once to create the vdi-base.ova
file. If this section was previously run, then it can be skipped unless
rebuild of the vdi-base.ova is desired.

**Note:** This section assumes no network connection is available. If the steps
above have been completed adequately, then the remainder of this procedure can
be completed offline with one exception.

1. Run the following to create the VM.

    $ common-inf/scripts/create-vbox-vm.sh ${VM_NAME} auto 512 10240 ${INSTALL_ISO_PATH}

1. Set Networking to Host-only Adapter:

    $ VBoxManage modifyvm ${VM_NAME} --nic1 hostonly --hostonlyadapter1 vboxnet0

1. Start the VM:

    $ VBoxManage startvm ${VM_NAME}

1. Follow the prompts:

  * Boot menu: Install
  * Select a language: English
  * Select your location: United States
  * Configure the keyboard: American English
  * Configure the network:
     * eth0
     * Continue without a default route? Yes
     * Name server addresses: (blank)
     * Hostname: $(HOST_NAME)
     * Domain name: (blank)
  * Set up users and passwords:
     * Root password: (blank)
     * Re-enter root password: (blank)
     * Full name: $(USER_FULLNAME)
     * Username: $(USERNAME)
     * Password: $(PASSWORD)
     * Re-enter password: $(PASSWORD)
  * Configure the clock: Central
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
     * Check SSH server **(IMPORTANT)**
     * Leave standard system utilities checked
     * Continue
  * Install the GRUB boot loader on a hard disk:
     * Yes
     * /dev/sda
  * Finish the installation: Continue

1. Remove any installation CDs or USB drives and reboot the VM. Do not power off
the VM at this point; let it continue to come up.

1. Set BIOS boot order as applicable to boot from the appropriate drive.


Verify Networking
================================================================================

**Note:** This section only needs to be run once to create the vdi-base.ova
file. If this section was previously run, then it can be skipped unless
rebuild of the vdi-base.ova is desired.

To find out the VM's IP address, log-in as $(USERNAME) from the VM console and
type:

    vdi$ /sbin/ifconfig -a

Verify eth0 has an IP address available: 192.168.56.___

From a terminal on the host, try to ssh into the VM using this IP address.

    host$ vdi_ip_addr=192.168.56.___
    host$ ssh ${USERNAME}@${vdi_ip_addr}
    Enter password:
    vdi$

From the VM, verify the host is available:

    vdi$ ssh user@192.168.56.1
    Enter password:
    host$ exit
    vdi$


Install Cloud-Init
================================================================================

**Note:** This section only needs to be run once to create the vdi-base.ova
file. If this section was previously run, then it can be skipped unless
rebuild of the vdi-base.ova is desired.

**Note:** This section needs the cloud-init-seed.iso image created in the
**Create cloud-init-seed.iso** section. Ensure that section is run prior to
continuing.

On the host:

    host$ VBoxManage storageattach ${VM_NAME} --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "${EXPORTED_VMS}/cloud-init-seed.iso"

Run the following in a terminal on the VDI guest:

(TODO clean-up)

    vdi$ sudo -i
    root# mount /media/cdrom0
    root# for p in \
       python-cheetah_2.4.4-3_amd64.deb \
       python-configobj_5.0.6-1_all.deb \
       python-jsonpatch_1.3-5_all.deb \
       python-oauth_1.0.1-4_all.deb \
       python-prettytable_0.7.2-3_all.deb \
       python-serial_2.6-1.1_all.deb \
       python-urllib3_1.9.1-3_all.deb \
       python-json-pointer_1.0-2_all.deb \
       libyaml-0-2_0.1.6-3_amd64.deb \
       python-requests_2.4.3-6_all.deb \
       python-boto_2.34.0-2_all.deb \
       python-yaml_3.11-2_amd64.deb \
       libpython3.4-minimal_3.4.2-1_amd64.deb \
       libmpdec2_2.4.1-1_amd64.deb \
       libpython3.4-stdlib_3.4.2-1_amd64.deb \
       libpython3-stdlib_3.4.2-2_amd64.deb \
       python3.4-minimal_3.4.2-1_amd64.deb \
       python3-minimal_3.4.2-2_amd64.deb \
       python3.4_3.4.2-1_amd64.deb \

       python3_3.4.2-2_amd64.deb \
       python3-apt_0.9.3.12_amd64.deb \
       dh-python_1.20141111-2_all.deb \
       unattended-upgrades_0.83.3.2+deb8u1_all.deb \
       python-software-properties_0.92.25debian1_all.deb \
       cloud-init_0.7.6~bzr976-2_all.deb;
    do
       dpkg --install /media/cdrom0/software/applications/ansible/${p};
    done



ansible_2.2.1.0-1ppa~trusty_all.deb
	cloud-init_0.7.6~bzr976-2_all.deb
	git_1%3a2.1.4-2.1_amd64.deb
	python-crypto_2.6.1-5+b2_amd64.deb
	python-httplib2_0.9+dfsg-2_all.deb
	python-jinja2_2.7.3-1_all.deb
	python-paramiko_1.15.1-1_all.deb
	python-setuptools_5.5.1-1_all.deb
	sshpass_1.05-1_amd64.deb


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

    host$ VBoxManage export ${VM_NAME} --output "${EXPORTED_VMS}/vdi-base.ova"

Remove the VM from the VirtualBox inventory. This will delete all files
pertaining to the VM, but it's OK since we can get them back from vdi-base.ova.

    host$ VBoxManage unregistervm ${VM_NAME} --delete


TODO Delete....
=================

Monitor progress (TODO if headless):

    host$ while [ 1 ]; do VBoxManage showvminfo ${VM_NAME} | grep State; sleep 30; done

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
