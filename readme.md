# Notes on Charliecloud

## Useful links

- documentation: [https://hpc.github.io/charliecloud](https://hpc.github.io/charliecloud])
- github repo: [https://github.com/hpc/charliecloud](https://github.com/hpc/charliecloud)

## CentOS 6 installation

### Install Docker

Latest Docker wants CentOS7

So install older from EPEL as described at
https://github.com/vfleaking/uoj/wiki/Install-Docker-on-CentOS-6.x

Start docker:
/etc/init.d/docker start

- follow installation instructions at https://hpc.github.io/charliecloud/install.html for Docker

### Install Charliecloud
```git clone --recursive https://github.com/hpc/charliecloud
```

Replace Makefile `CFLAGS -std=c11` to `-std=c99`
```
$ make SETUID=yes
```
Since using SETUID, it wants to chown ch-run to root, do that manually
```
$ chown root /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/charliecloud/bin/ch-run
$ chmod u+s /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/charliecloud/bin/ch-run
```
Then install as root:
```
$ cd /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/charliecloud
$ make install PREFIX=/uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/std
```

As user (not as root), try
```
$ /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/charliecloud/bin/ch-run --version
```

Proceed to testing, 
https://hpc.github.io/charliecloud/install.html#test-charliecloud

Need to make sure that user has sudo...

As user
```
$ export PATH=/uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/std/bin:$PATH
$ export CH_TEST_TARDIR=/var/tmp/tarballs
$ export CH_TEST_IMGDIR=/var/tmp/images
$ export CH_TEST_PERMDIRS='/var/tmp /tmp'
```

Then go to https://hpc.github.io/charliecloud/virtualbox.html#test and run the tests

Note that sudo is needed for running the tests, but not for running finished containers.

## CentOS7 installation

First make sure that user namespaces are enabled in CentOS7:
```
$ grubby --args="namespace.unpriv_enable=1" --update-kernel="$(grubby --default-kernel)"
```
and creating file `/etc/sysctl.d/51-userns.conf` with the following in it:
```
user.max_user_namespaces = 32767
```
### The installation (done on singularity.chpc.utah.edu)

Follow installation instructions at https://hpc.github.io/charliecloud/install.html 

#### Docker setup 
```
$ sudo yum install yum-utils
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum install docker-ce
$ sudo systemctl enable docker
$ sudo systemctl is-enabled docker
$ sudo systemctl start docker
```
then test to make sure we can run container:
```
$ sudo docker run hello-world
```
#### Charliecloud setup
```
$ sudo vi /etc/profile.d/charliecloud.sh
export CH_TEST_TARDIR=/var/tmp/tarballs
export CH_TEST_IMGDIR=/var/tmp/images
export CH_TEST_PERMDIRS=skip

$ sudo ln -s /usr/lib/systemd/system/getty@.service /etc/systemd/system/getty.target.wants/getty@tty2.service
$ sudo systemctl start getty@tty2.service

$ cd /usr/local/src
$ git clone --recursive https://www.github.com/hpc/charliecloud.git
$ cd charliecloud
$ make
$ sudo make install PREFIX=/usr
```

### Some docker commands

`sudo docker images` - list created images
`sudo docker build -t hello .` - build Docker container

### Building sample container
```
cd /home/u0101881/charliecloud/charliecloud/examples/serial/hello sample
ch-build -t hello /home/u0101881/charliecloud/charliecloud/examples/serial/hello
ch-docker2tar hello /var/tmp
ch-tar2dir /var/tmp/hello.tar.gz /var/tmp
```
### Simple running
```
ch-run /var/tmp/hello -- echo hello
ch-run /var/tmp/hello -- bash
```
- file systems - need to include source:destination, otherwise destination is set to /mnt/[0-9]
```
ch-run -b /uufs:/uufs -b /scratch:/scratch /var/tmp/hello -- bash
```

## Quick setup on scrubpeak

### running as an user with SETUID
```
setenv PATH /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/std/bin:$PATH
```
### running as and user with user namespaces
```
setenv PATH /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/namespace-oct17/bin:$PATH
cd /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers
ch-tar2dir hello.tar.gz /var/tmp
ch-run -b /uufs:/uufs -b /scratch:/scratch /var/tmp/hello -- bash
```

## MPI container with InfiniBand
### Container setup and testing on scrubpeak (w/o IB)
#### On singularity:
```
cd /home/u0101881/charliecloud/containers/ubuntu1604openmpi3
ch-build -t ubuntu1604openmpi3 /home/u0101881/charliecloud/containers/ubuntu1604openmpi3
ch-docker2tar ubuntu1604openmpi3 /var/tmp
scp /var/tmp/ubuntu1604openmpi3.tar.gz u0101881@scrubpeak:/uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers
```
#### On scrubpeak:
Build OpenMPI w/o PSM and SLURM - openmpi/3.0.0.g

Then from scrubpeak interactive
```
srun -N 2 -n 16 -A chpc -p scrubpeak -t 1:00:00 --pty /bin/bash -l
cd srun -N 2 -n 16 -A chpc -p scrubpeak -t 1:00:00 --pty /bin/bash -l
ml gcc openmpi/3.0.0.g
mpicc hello_mpi.c
mpirun -np 16 a.out

export PATH=/uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/namespace-oct17/bin:$PATH
cd /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers
ch-tar2dir ubuntu1604openmpi3.tar.gz /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers/temp/
ch-run -b /uufs:/uufs -b /scratch:/scratch /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers/temp/ubuntu1604openmpi3 -- bash
ldd ~u0101881/tests/a.out # liblustreapi.so is not resolved since host built OMPI autodetects lustre
cd ~u0101881/tests
ml purge # lmod is in the container so if openmpi/3.0.0.g is loaded it is trying to build with that
mpicc hello_mpi.c -o a.out.chc
mpirun -np 16 ./a.out.chc # running inside of the container does not work as container OpenMPI was not built with SLURM or PMIx
exit # out of the container
# now on the host
mpirun --mca btl tcp,self -np 16 ch-run -b /uufs:/uufs -b /scratch:/scratch /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers/temp/ubuntu1604openmpi3 -- ~u0101881/tests/a.out.chc # works
```

### Deployment on the IB clusters (initially Ember GPU nodes)

Make sure that user namespaces are enabled

Then submit interactive job:
```
$ srun -N 2 -n 2 -A ember-gpu -p ember-gpu -t 1:00:00 --pty /bin/bash -l
$ export PATH=/uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/namespace-oct17/bin:$PATH
```

compile the MPI program in the container:
``
$ ch-run -b /uufs:/uufs -b /scratch:/scratch /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers/temp/ubuntu1604openmpi3 -- bash
$ cd
$ cd ~u0101881/tests
$ /usr/bin/mpicc hello_mpi.c
```

Then run outside of container:
```
$ mpirun -np 2 ch-run -b /uufs:/uufs -b /scratch:/scratch /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers/temp/ubuntu1604openmpi3 -- /uufs/chpc.utah.edu/common/home/u0101881/tests/a.out
Hello world from processor em001, rank 0 out of 2 processors
Hello world from processor em012, rank 1 out of 2 processors
```
#### Container with IMB
- base on ubuntu1604openmpi3, built on `singularity.chpc.utah.edu` in `/home/u0101881/charliecloud/containers/mpibench`
```
$ ch-build -t mpibench /home/u0101881/charliecloud/containers/mpibench
$ ch-docker2tar mpibench /var/tmp
$ scp /var/tmp/mpibench.tar.gz u0101881@scrubpeak:/uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers
```

- run on Ember
```
$ srun -N 2 -n 2 -A ember-gpu -p ember-gpu -t 1:00:00 --pty /bin/bash -l
$ export PATH=/uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/namespace-oct17/bin:$PATH
$ mpirun -np 2 ch-run -b /uufs:/uufs -b /scratch:/scratch /uufs/chpc.utah.edu/common/home/u0101881/containers/charliecloud/containers/temp/mpibench -- /usr/src/imb/src/IMB-MPI1
...
#---------------------------------------------------
# Benchmarking PingPong 
# #processes = 2 
#---------------------------------------------------
       #bytes #repetitions      t[usec]   Mbytes/sec
            0         1000         1.73         0.00
...
      4194304           10      1760.17      2272.51
```

## Charliecloud vs. Singularity

### Positives
- very simple and presumably potential priviledge escalation free (especially when using user namespaces)
- uses Docker to build containers
- cached container building using Docker layers - no need to rebuild a whole container in order to add a few lines to the def file

### Negatives
- more complicated container deployment
 -- Docker local image repository is confusing to navigate in
 -- need to run a few commands to extract the container image from the Docker repo into a tar.gz file
- does not seem to allow expanding the container into NFS mounted file system - the container needs to be expanded ot local drive

## Things to note
 - in the SETUID install version, the ch-run does not include the --uid and --gid flags


## A side note on MPI program launching via SLURM

As of Oct17 (SLURM 17.02 and recent MPIs), all mpiruns are launched with srun underneath, using the PMI that comes with that respective MPI.

For example, MPICH 2 processes, one on each node:
node 0:
u0101881 22732 20560  0 17:15 pts/0    00:00:00 mpirun -v -np 2 ./a.out
u0101881 22733 22732  0 17:15 ?        00:00:00 /uufs/scrubpeak.peaks/sys/pkg/slurm/std/bin/srun -N 2 -n 2 --input none /uufs/chpc.utah.edu/sys/installdir/mpich/3.2-c7/bin/hydra_pmi_proxy --control-port sp001:44787 --debug --rmk slurm --launcher slurm --demux poll --pgid 0 --retries 10 --usize -2 --proxy-id -1
u0101881 22734 22733  0 17:15 ?        00:00:00 /uufs/scrubpeak.peaks/sys/pkg/slurm/std/bin/srun -N 2 -n 2 --input none /uufs/chpc.utah.edu/sys/installdir/mpich/3.2-c7/bin/hydra_pmi_proxy --control-port sp001:44787 --debug --rmk slurm --launcher slurm --demux poll --pgid 0 --retries 10 --usize -2 --proxy-id -1
u0101881 22747 22742  0 17:15 ?        00:00:00 /uufs/chpc.utah.edu/sys/installdir/mpich/3.2-c7/bin/hydra_pmi_proxy --control-port sp001:44787 --debug --rmk slurm --launcher slurm --demux poll --pgid 0 --retries 10 --usize -2 --proxy-id -1
u0101881 22758 22747 96 17:15 ?        00:00:15 ./a.out

node 1:
u0101881 20929 20924  0 17:15 ?        00:00:00 /uufs/chpc.utah.edu/sys/installdir/mpich/3.2-c7/bin/hydra_pmi_proxy --control-port sp001:44787 --debug --rmk slurm --launcher slurm --demux poll --pgid 0 --retries 10 --usize -2 --proxy-id -1
u0101881 20938 20929 99 17:15 ?        00:01:04 ./a.out

OpenMPI:
node 0:
u0101881 21651 20560  0 16:56 pts/0    00:00:00 mpirun --mca btl tcp,self -vvv -np 2 ./a.out
u0101881 21656 21651  0 16:56 pts/0    00:00:00 srun --ntasks-per-node=1 --kill-on-bad-exit --cpu_bind=none --nodes=1 --nodelist=sp002 --ntasks=1 orted -mca ess "slurm" -mca ess_base_jobid "3187605504" -mca ess_base_vpid "1" -mca ess_base_num_procs "2" -mca orte_node_regex "sp[3:1-2]@0(2)" -mca orte_hnp_uri "3187605504.0;tcp://10.242.10.1:54531" --mca btl "tcp,self"
u0101881 21657 21656  0 16:56 pts/0    00:00:00 srun --ntasks-per-node=1 --kill-on-bad-exit --cpu_bind=none --nodes=1 --nodelist=sp002 --ntasks=1 orted -mca ess "slurm" -mca ess_base_jobid "3187605504" -mca ess_base_vpid "1" -mca ess_base_num_procs "2" -mca orte_node_regex "sp[3:1-2]@0(2)" -mca orte_hnp_uri "3187605504.0;tcp://10.242.10.1:54531" --mca btl "tcp,self"
u0101881 21663 21651 99 16:56 pts/0    00:01:10 ./a.out
node 1:
u0101881 19748 19743  0 16:56 ?        00:00:00 /uufs/chpc.utah.edu/sys/installdir/openmpi/3.0.0-c7/bin/orted -mca ess "slurm" -mca ess_base_jobid "3187605504" -mca ess_base_vpid "1" -mca ess_base_num_procs "2" -mca orte_node_regex "sp[3:1-2]@0(2)" -mca orte_hnp_uri "3187605504.0;tcp://10.242.10.1:54531" --mca btl "tcp,self"
u0101881 19760 19748 99 16:56 ?        00:00:46 ./a.out


