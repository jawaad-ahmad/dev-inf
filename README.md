dev-inf
================================================================================

Infrastructure as code for development projects


Introduction
================================================================================

This document sets up an initial VM environment to bootstrap a local
development VDI or deployment server.

Important: The entire point of this is to be able to set up a basic starting
point VM from scrach without the Ansible environment until Ansible can take
over; therefore, all steps must be performed manually.

Running through this document will provide you with a VM OVA file and a
cloud-init ISO file that you can you repeatedly to quickly generate a
development VDI or deployment server as often as desired from a base
configuration maintained in a git repository.

The _Provision Base Instance_ section provides instructions on how to use a
pre-built OVA and ISO file to create and start a VM instance. The remainder of
the document describes the process for creating the OVA and ISO files.


References
================================================================================

  * https://medium.com/@mvuksano/automated-provisioning-of-virtual-machines-for-development-8a543e435f44
  * https://vladimir-ivanov.net/how-to-compact-virtualboxs-vdmk-file-size/


Variables
================================================================================

The following variables are used throughout this procedure:

   | Key               | Value           |
   |-------------------|-----------------|
   | INSTALL_ISO_PATH  | /path/to/downloaded/Debian/jessie/debian-8.5.0-amd64-DVD-1.iso |
   | EXPORTED_VMS      | /path/to/ova    |
   | ANSIBLE_DATA_HOME | /path/to/files  |
   | VM_NAME           | vdi -or- deploy |
   | HOST_NAME         | ${VM_NAME}      |
   | USER_FULLNAME     | Ansible         |
   | USERNAME          | ansible         |
   | PASSWORD          | ansible         |


Provision Base Instance
================================================================================

Using the base OVA and the cloud-init seed.iso built as described further below,
this section provisions a new VDI or deployment server.

Begin by importing the OVA and attaching the ISO as its CD-ROM drive:

    host$ base=vdi-base.ova

      -or-

    host$ base=deploy-base.ova

Then:

    host$ VBoxManage import "${EXPORTED_VMS}/${base}" --vsys 0 --vmname ${VM_NAME} && VBoxManage storageattach ${VM_NAME} --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "${EXPORTED_VMS}/cloud-init/seed.iso"

Start the VM:

    host$ VBoxManage startvm ${VM_NAME}

Do not do anything with the VM while it is up for now; however, if you wish to
monitor what is going on, you can log-in via the console or via SSH and monitor
the cloud-init output log:

   host$ ssh ansible@vdi
   vdi$ sudo tail -F /var/log/cloud-init-output.log

Wait for the VM to power itself down. This should take around 10 minutes.

If there is any trouble, run the following to destroy and recreate the VM:

    host$ VBoxManage controlvm ${VM_NAME} poweroff; VBoxManage unregistervm ${VM_NAME} --delete; sleep 2; VBoxManage import "${EXPORTED_VMS}/${base}" --vsys 0 --vmname ${VM_NAME} && VBoxManage storageattach ${VM_NAME} --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "${EXPORTED_VMS}/cloud-init/seed.iso" && VBoxManage startvm ${VM_NAME}

Note that the command above will power down the VM hard. Ideally, it would be
nicer to run something like sudo poweroff from inside the VM. If you get a DHCP
lease, this might help to preserve your assigned IP address while
troubleshooting depending on your router.


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

Begin by downloading the desired Debian ISO for installing Debian. Save this as
${INSTALL_ISO_PATH}.

Then, clone the git repo containing the development infrastructure.

    $ cd ~/repo
    $ git clone https://github.com/jawaad-ahmad/dev-inf
    $ cd ~/repo/dev-inf
    $ ./init.sh


Create cloud-init seed.iso
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
    $ /bin/rm -Rf "${CI_HOME}"
    $ mkdir -p "${CI_HOME}"

Ensure your ${ANSIBLE_DATA_HOME} contains the following:

    $ find ${ANSIBLE_DATA_HOME} -type f
    software/applications/ansible/ansible_2.7.1-1ppa~trusty_all.deb
    software/applications/ansible/cloud-init_0.7.6~bzr976-2_all.deb
TODO Delete? {
    software/applications/ansible/dh-python_1.20141111-2_all.deb
    software/applications/ansible/git_1%3a2.1.4-2.1_amd64.deb
    software/applications/ansible/libmpdec2_2.4.1-1_amd64.deb
    software/applications/ansible/libpython3-stdlib_3.4.2-2_amd64.deb
    software/applications/ansible/libpython3.4-stdlib_3.4.2-1_amd64.deb
    software/applications/ansible/libpython3.4-minimal_3.4.2-1_amd64.deb
    software/applications/ansible/libyaml-0-2_0.1.6-3_amd64.deb
    software/applications/ansible/python3.4_3.4.2-1_amd64.deb
    software/applications/ansible/python3-apt_0.9.3.12_amd64.deb
    software/applications/ansible/python-configobj_5.0.6-1_all.deb
    software/applications/ansible/python3_3.4.2-2_amd64.deb
    software/applications/ansible/python-json-pointer_1.0-2_all.deb
    software/applications/ansible/python-jsonpatch_1.3-5_all.deb
    software/applications/ansible/python-crypto_2.6.1-5+b2_amd64.deb
    software/applications/ansible/python-serial_2.6-1.1_all.deb
    software/applications/ansible/python-prettytable_0.7.2-3_all.deb
    software/applications/ansible/python-setuptools_5.5.1-1_all.deb
    software/applications/ansible/python3-minimal_3.4.2-2_amd64.deb
    software/applications/ansible/python3.4-minimal_3.4.2-1_amd64.deb
    software/applications/ansible/python-boto_2.34.0-2_all.deb
    software/applications/ansible/python-cheetah_2.4.4-3_amd64.deb
    software/applications/ansible/python-httplib2_0.9+dfsg-2_all.deb
    software/applications/ansible/python-jinja2_2.7.3-1_all.deb
    software/applications/ansible/python-oauth_1.0.1-4_all.deb
    software/applications/ansible/python-paramiko_1.15.1-1_all.deb
    software/applications/ansible/python-requests_2.4.3-6_all.deb
    software/applications/ansible/python-software-properties_0.92.25debian1_all.deb
    software/applications/ansible/python-urllib3_1.9.1-3_all.deb
    software/applications/ansible/python-yaml_3.11-2_amd64.deb
    software/applications/ansible/sshpass_1.05-1_amd64.deb
    software/applications/ansible/unattended-upgrades_0.83.3.2+deb8u1_all.deb
}
    software/applications/node-v6.10.1-linux-x64.tar.xz

Copy the cloud-init files:

    $ cp -a ~/repo/dev-inf/cloud-init/* "${CI_HOME}"
    $ mkdir -p ${CI_HOME}/software/applications
    $ cp -a {${ANSIBLE_DATA_HOME},${CI_HOME}}/software/applications/ansible

Generate the cloud-init seed ISO:

    $ mkdir -p "${EXPORTED_VMS}/cloud-init"
    $ genisoimage -output "${EXPORTED_VMS}/cloud-init/seed.iso" -volid cidata -R -J "${CI_HOME}"


Bootstrap Base Install
================================================================================

**Note:** This section only needs to be run once to create the base OVA files.
If this section was previously run, then it can be skipped unless rebuild of
the base OVA(s) is desired.

**Note:** This section assumes no network connection is available. If the steps
above have been completed adequately, then the remainder of this procedure can
be completed offline with one exception.

**Note:** Running this section should take approximately 35 minutes.

Run the following to create the VM.

    $ common-inf/scripts/create-vbox-vm.sh ${VM_NAME} auto 512 10240 ${INSTALL_ISO_PATH}

Start the VM:

    $ VBoxManage startvm ${VM_NAME}

Follow the prompts:

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
     * Check or uncheck Debian desktop environment:
        * **For a VDI image:** You can check this so the initial install is
          faster by installing packages initally from the local installation
          media rather than downloading from the Internet; any updates will be
          installed later anyway.
        * **For a deployment server:** You will not want a graphical
          environment, so make sure to uncheck this.
     * Uncheck print server
     * Check SSH server **(IMPORTANT)**
     * Leave standard system utilities checked
     * Continue
  * Install the GRUB boot loader on a hard disk:
     * Yes
     * /dev/sda
  * Finish the installation: Continue

Remove any installation CDs or USB drives and reboot the VM. Do not power off
the VM at this point; let it continue to come up unless...

If desired, this would be a good point to power off the VM and take a snapshot
if experimenting in order to restore back to this snapshot later.

Set BIOS boot order as applicable to boot from the appropriate drive.


Verify Networking
================================================================================

**Note:** This section only needs to be run once to create the base OVA file.
If this section was previously run, then it can be skipped unless rebuild of
the base OVA(s) is desired.

To find out the VM's IP address, log-in as $(USERNAME) from the VM console and
type:

    vdi$ /sbin/ifconfig -a

Verify eth0 has an IP address available: 192.168.___.___

From a terminal on the host, try to ssh into the VM using this IP address.

    host$ vdi_ip_addr=192.168.___.___
    host$ ssh ${USERNAME}@${vdi_ip_addr}
    Enter password:
    vdi$


Install Cloud-Init and Ansible
================================================================================

**Note:** This section only needs to be run once to create the base OVA file.
If this section was previously run, then it can be skipped unless rebuild of
the base OVA(s) is desired.

**Note:** This section needs the cloud-init seed.iso image created in the
**Create cloud-init seed.iso** section. Ensure that section is run prior to
continuing.

On the host:

    host$ VBoxManage storageattach ${VM_NAME} --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "${EXPORTED_VMS}/cloud-init/seed.iso"

Run the following in a terminal on the VDI guest:

    vdi$ sudo -i

Begin by setting up the /etc/role file. The cloud-init user-data script will
use this to determine which Ansible playbook to run.

    root# echo vdi > /etc/role

      -or-

    root# echo deployment > /etc/role

Continue:

    root# mount -o ro /media/cdrom0
    root# sed --in-place=.orig -e 's/^\(deb\ cdrom\:\)/\#\1/' /etc/apt/sources.list
    root# echo deb ftp://ftp.us.debian.org/debian jessie main non-free contrib > /etc/apt/sources.list.d/tempsrc.list
    root# apt-get update
    root# apt-get install --assume-yes cloud-init git python-crypto python-httplib2 python-jinja2 python-paramiko python-setuptools sshpass
    root# dpkg --install /media/cdrom0/software/applications/ansible/ansible_2.7.1-1ppa~trusty_all.deb


Export Base VM
================================================================================

**Note:** This section only needs to be run once to create the base OVA file.
If this section was previously run, then it can be skipped unless rebuild of
the base OVA(s) is desired.

Run the following in a terminal on the VDI/deployment host to clean up the VM
hard drive and then power off the VM:

    root# /bin/rm /etc/apt/sources.list.d/tempsrc.list
    root# /bin/mkdir /tmp2
    root# for s in export-prep.sh zero-free-space.sh; do wget -O /tmp2/${s} "https://raw.githubusercontent.com/jawaad-ahmad/common-inf/master/scripts/${s}"; chmod +x /tmp2/${s}; done
    root# /tmp2/export-prep.sh

TODO Observed the following messages and errors while running this:

         (This section is currently unimplemented.)
         ...
         (This section is currently unimplemented.)
         ...
         Determining file name for writing. Checking //zf0...
         Found no file //zf0
         Writing  blocks to //zf0 (deploy only, not vdi)
         /bin/dd: invalid number ‘’ (deploy only, not vdi)

    root# for f in /tmp2/*; do shred --iterations=1 --zero ${f}; done
    root# /bin/rm -Rf /tmp2
    root# poweroff

(TODO Not sure about the re-thinning yet; didn't check to see the original
VMDK file size before exporting; simply noticed the imported VMDK was a
whopping 4 GB. Need to go through this one more time to make sure this is
useful, because at the moment it didn't do anything to reduce the file size.)

The export-prep.sh script filled the virtual hard drive free space with zeroes,
but Virtual Box exporting doesn't appear to be great at excluding those in the
OVA. Let's compact the hard drive before the export on the host:

    host$ cd ${HOME}/VirtualBox\ VMs/${VM_NAME}
    host$ vdiDiskVmdk=vdi-base-disk1.vmdk
    host$ vdiDiskVdi=vdi-base-disk1.vdi
    host$ hddUuid="$(VBoxManage showhdinfo vdi-base-disk1.vmdk  | awk -F ' ' '/^UUID\:/ { print $2; }')"

Convert the virtual hard drive to VDI format, compact it, and then convert it
back to VMDK format:

    host$ VBoxManage clonehd ${vdiDiskVmdk} ${vdiDiskVdi} --format vdi
    host$ VBoxManage modifyhd ${vdiDiskVdi} --compact
    host$ /bin/rm ${vdiDiskVmdk}
    host$ VBoxManage clonehd ${vdiDiskVdi} ${vdiDiskVmdk} --format vmdk
    host$ /bin/rm ${vdiDiskVdi}

Finally, set the original UUID:

    host$ VBoxManage internalcommands sethduuid ${vdiDiskVmdk} ${hddUuid}

Now, depending on what you're building...

    host$ base=vdi-base.ova

      -or-

    host$ base=deploy-base.ova

...run the following to create the base VM OVA:

    host$ VBoxManage export ${VM_NAME} --output "${EXPORTED_VMS}/${base}"

Remove the VM from the VirtualBox inventory. This will delete all files
pertaining to the VM, but it's OK since we can get them back from base OVA.

    host$ VBoxManage unregistervm ${VM_NAME} --delete


TODO Delete....
=================

{
{
The rest of this document isn't needed; it can be deleted.

Set Networking to Host-only Adapter: (TODO won't work - we need bridged later for Internet access when downloading cloud-init)

    $ VBoxManage modifyvm ${VM_NAME} --nic1 hostonly --hostonlyadapter1 vboxnet0

From the VM, verify the host is available:

    vdi$ ssh user@192.168.56.1
    Enter password:
    host$ exit
    vdi$


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

TODO...

   pxe-servers:late-command.sh
   $ VBoxManage controlvm ${VM_NAME} poweroff; VBoxManage unregistervm ${VM_NAME} --delete; sleep 2; /bin/rm -Rf ${CI_HOME}; cp -a ~/repo/dev-inf/cloud-init ${CI_HOME} && mkdir -p ${CI_HOME}/software && cp -a ${ANSIBLE_DATA_HOME}/software/applications ${CI_HOME}}/software && genisoimage -output "${EXPORTED_VMS}/cloud-init/seed.iso" -volid cidata -R -J "${CI_HOME}" && VBoxManage import "${EXPORTED_VMS}/vdi-base.ova" --vsys 0 --vmname ${VM_NAME} && VBoxManage storageattach ${VM_NAME} --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "${EXPORTED_VMS}/cloud-init/seed.iso" && VBoxManage startvm ${VM_NAME}
   $ for ip in 15; do ssh-keygen -f "/home/jawaad/.ssh/known_hosts" -R 192.168.1.${ip} && ssh-keyscan -H 192.168.1.${ip} >> ~/.ssh/known_hosts && ssh ansible@192.168.1.${ip}; done


Install SSH Authorized Key
================================================================================

    host$ cat ~/.ssh/id_rsa.pub | ssh ansible@${vdi_ip_addr} 'cat - >> ~/.ssh/authorized_keys'
    Enter password:
    host$ ssh ansible@${vdi_ip_addr}
    vdi$

Verify logged-in with no password prompt.
}

