-----------
Description
-----------

Installs and configures Lustre 2.8.0 server and clients.
The configuration is a simplified setup inspired by
http://www.dell.com/downloads/global/solutions/200-DELL-CAMBRIDGE-SOLUTIONS-WHITEPAPER-20072010b.pdf
     
                -----------------------------
                |       MGS/MDS Module      |
                |                           |
                |        -----------        |
                |MGT01 <-|         |->      |
                |MDT01 <-| Shared  |->      |
                |  |     | Storage |     |  |
                |  |     |         |     |  |
                |  |     -----------     |  |
                |  |                     |  |
                | -----------   ----------- |
                | |  MDS01  |   |  MDS02  | |
                | |  Active |   | Standby | |
                | -----------   ----------- |
                -----------------------------
                              | 
    -----------               |
    |         |               |
    | Clients |               |                -----------------------------
    |         | -------------------------------|        OSS Module         | 
    -----------            NETWORK             |                           |
                                               |        -----------        |
                                               |OST01 <-|         |->      |
                                               |  |   <-| Shared  |-> OST02| 
                                               |  |     | Storage |     |  |
                                               |  |     |         |     |  |
                                               |  |     -----------     |  |
                                               |  |                     |  |
                                               | -----------   ----------- |
                                               | |  OSS01  |   |  OSS02  | |
                                               | |  Active |   |  Active | |
                                               | -----------   ----------- |
                                               -----------------------------

Server supported only on CentOS 6.
Clients supported on CentOS 6/7.

For introduction to lustre see https://www.citutor.org/login.php?course=61
"Using the Lustre File System - Cyberinfrastructure Tutor"
and https://wiki.hpdd.intel.com/display/PUB

------------
Sample Usage
------------

Assuming you have VirtualBox and Vagrant installed
https://www.virtualbox.org/ https://www.vagrantup.com/downloads.html::

        $ git clone https://github.com/marcindulak/vagrant-lustre-tutorial.git
        $ cd vagrant-lustre-tutorial
        $ vagrant up mds01 && sleep 10 && vagrant reload mds01
        $ vagrant up mds02 && sleep 10 && vagrant reload mds02
        $ vagrant up oss01 && sleep 10 && vagrant reload oss01
        $ vagrant up oss02 && sleep 10 && vagrant reload oss02
        $ vagrant up centos7 && sleep 10 && vagrant reload centos7

*Note*: vagrant-reload plugin is no longer maintained so we cannot reload a VM from within Vagrantfile
(see https://github.com/aidanns/vagrant-reload/issues/6) and we need to use `up` followed by `reload` explicitly.

Test the basic funcionality of the lustre filesystem with, e.g.::

        $ vagrant ssh centos7 -c "sudo su -c 'lctl dl'"
        $ vagrant ssh centos7 -c "sudo su -c 'lfs osts'"
        $ vagrant ssh centos7 -c "sudo su -c 'dd if=/dev/zero of=/lustre/testfile bs=1M count=512'"
        $ vagrant ssh centos7 -c "sudo su -c 'lfs getstripe /lustre'"
        $ vagrant ssh centos7 -c "sudo su -c 'lfs find  /lustre -type f -print'"
        $ vagrant ssh centos7 -c "sudo su -c 'lfs df -h'"
        $ vagrant ssh centos7 -c "sudo su -c 'rm -f /lustre/testfile'"
        $ vagrant ssh centos7 -c "sudo su -c 'mkdir /lustre/stripe_2'"
        $ vagrant ssh centos7 -c "sudo su -c 'lfs setstripe -c 2 /lustre/stripe_2'"
        $ vagrant ssh centos7 -c "sudo su -c 'dd if=/dev/zero of=/lustre/stripe_2/testfile bs=1M count=512'"
        $ vagrant ssh centos7 -c "sudo su -c 'lfs getstripe /lustre/stripe_2/testfile'"
        $ vagrant ssh centos7 -c "sudo su -c 'rm -rf /lustre/stripe_2'"

Benchmark with IOR, see http://goo.gl/7AWwQ (https://docs.google.com/document/d/1mTXXXYadO7QLVqs29BtzPzoffwKsYoVgwBS_s_pGxnU) referenced at http://wiki.opensfs.org/Zero_to_Hero.
Testing file-per-process (only one ior process used here) small, random IO, assuming the server has 256MB RAM::

        $ vagrant ssh centos7 -c "sudo su -c 'wget https://github.com/chaos/ior/archive/3.0.1.tar.gz -O ior-3.0.1.tar.gz'"
        $ vagrant ssh centos7 -c "sudo su -c 'tar zxf ior-3.0.1.tar.gz'"
        $ vagrant ssh centos7 -c "sudo su -c 'yum -y install automake'"
        $ vagrant ssh centos7 -c "sudo su -c 'cd ior-3.0.1; sh bootstrap'"
        $ vagrant ssh centos7 -c "sudo su -c 'yum -y install openmpi-devel'"
        $ vagrant ssh centos7 -c "sudo su -c '. /etc/profile.d/modules.sh; module load mpi; cd ior-3.0.1; ./configure'"
        $ vagrant ssh centos7 -c "sudo su -c '. /etc/profile.d/modules.sh; module load mpi; cd ior-3.0.1; make'"
        $ vagrant ssh centos7 -c "sudo su -c '. /etc/profile.d/modules.sh; module load mpi; mpirun --allow-run-as-root -np 1 --bynode ior-3.0.1/src/ior -v -a POSIX -i5 -g -e -w -r 512m -b 4m -o /lustre/testfile -F -C -b 256k -t 4k -O lustreStripeCount=1 -z random'"

Other benchmarks from http://www.opensfs.org/wp-content/uploads/2013/04/LIND_LUG_2013.pdf::

        $ vagrant ssh centos7 -c "sudo su -c 'yum -y install bonnie++'"
        $ vagrant ssh centos7 -c "sudo su -c 'bonnie++ -d /lustre -r 256 -u root'"
        $ vagrant ssh centos7 -c "sudo su -c 'yum -y install http://pkgs.repoforge.org/rpmforge-release/rpmforge-release-0.5.3-1.el7.rf.x86_64.rpm'"
        $ vagrant ssh centos7 -c "sudo su -c 'yum -y install iozone'"
        $ vagrant ssh centos7 -c "sudo su -c 'iozone –w –M –t 1 –s 512m –r 1m -i 0 –i 1 –F /lustre -R'"
        $ vagrant ssh centos7 -c "sudo su -c 'wget http://downloads.sourceforge.net/project/mdtest/mdtest%20latest/mdtest-1.9.3/mdtest-1.9.3.tgz'"
        $ vagrant ssh centos7 -c "sudo su -c 'mkdir mdtest-1.9.3; cd mdtest-1.9.3'"
        $ vagrant ssh centos7 -c "sudo su -c 'cd mdtest-1.9.3; tar zxf ../mdtest-1.9.3.tgz'"
        $ vagrant ssh centos7 -c "sudo su -c 'sed -i \"s/OS=.*/OS=Linux/\" mdtest-1.9.3/Makefile'"
        $ vagrant ssh centos7 -c "sudo su -c 'yum -y install openmpi-devel'"
        $ vagrant ssh centos7 -c "sudo su -c '. /etc/profile.d/modules.sh; module load mpi; cd mdtest-1.9.3; MPI_CC=mpicc make'"
        $ vagrant ssh centos7 -c "sudo su -c '. /etc/profile.d/modules.sh; module load mpi; ./mdtest-1.9.3/mdtest –I 10 –i 5 –z 5 –b 2 –d /lustre'"
        $ vagrant ssh centos7 -c "sudo su -c 'dd if=/dev/zero of=/lustre/testfile bs=1M count=512 oflag=direct conv=fdatasync'"
        $ vagrant ssh centos7 -c "sudo su -c 'rm -f /lustre/testfile'"

Test other clients::

        $ vagrant up centos6 && sleep 10 && vagrant reload centos6
        $ vagrant ssh centos6 -c "sudo su -c 'lfs df -h'"
        $ # vagrant ssh centos6_lustre18 -c "sudo su -c 'lfs df -h'"  # this build seems not working with lustre-2.8: mds01 says "... is it running a compatible version of Lustre ..."
        $ # vagrant ssh ubuntu12 -c "sudo su -c 'lfs df -h'"  # hangs with lustre-2.7
        $ # vagrant ssh ubuntu12 -c "sudo su -c 'lfs df -h'"  # interestingly lustre-utils 1.8.5 seems to work still with lustre-2.8 ...

Test vagrant user access to the filesystem::

        $ vagrant ssh centos6 -c "sudo su -c 'mkdir /lustre/vagrant'"
        $ vagrant ssh centos6 -c "sudo su -c 'chown vagrant.vagrant /lustre/vagrant'"
        $ vagrant ssh centos6 -c "sudo su - vagrant -c 'touch /lustre/vagrant/testfile'"

*Note*: users' uid/gid on the client must agree with those on mds.

Test (manual) failover of an OST::

        $ vagrant ssh oss02 -c "sudo su -c 'umount /lustre/ost02'"
        $ vagrant ssh centos7 -c "sudo su -c 'lfs df -h'"  # this will hang
        $ vagrant ssh oss01 -c "sudo su -c 'echo \"/dev/sdc /lustre/ost02 lustre defaults 0 0\" >> /etc/fstab'"
        $ vagrant ssh oss01 -c "sudo su -c 'mkdir -p /lustre/ost02'"
        $ vagrant ssh oss01 -c "sudo su -c 'mount /lustre/ost02'"
        $ vagrant ssh centos7 -c "sudo su -c 'lfs df -h'"

When done, destroy the test machines with::

        $ vagrant destroy -f


------------
Dependencies
------------

None


-------
License
-------

BSD 2-clause


----
Todo
----

