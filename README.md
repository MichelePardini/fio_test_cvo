# fio_test_cvo

## This REPO contains a set of .fio configuration files that can be used to run performance test on a NetApp Cloud Volumes ONTAP.

This are the pre-req when configuring the testbed.
1) When you deploy a CVO from Cloud Manager, the configuration should be
  - 2 data aggregates, each composed of 6 EBS disk of 4TB each. Total of 12 disks that should allow 3GB/s max tput, or 144K op/s at 16Kb
  - Each aggregate contains 2 volumes of 2TB with Storage Efficiencies disabled. So 4 volumes in total
  - 8 clients running RHEL 7.7. 2 RHEL per volume, mounting the export with default parameters. You can use CenOS as well.
  - The Working Set Size (WSS) varies depending on the instance type, the main rules are
      - Twice the size of the Memory (RAM) if the instance does not have an NVME FlashCache
      - Twice the size of the FlashCache, if present
      - The resulting WSS will be divided /4 volumes
      
2) Use FIO v3.16 min.

## How to use it

a) Decide the size of the WSS based on the instance type<br/>
b) edit the 'create_dataset.fio' and change the params 'size' and 'nrfiles' to accomodate the new WSS. Avoid changing 'numjobs'<br/>
c) run 'fio create_dataset.fio' from 4 clients to create the dataset on the 4 volumes<br/>
d) once the dataset has been created, decide which workload to run. For seq rx you can run 'fio 64k_seq_wr.fio' from all 8 Linux at the same time<br/>

When running the test you can also capture a perfstat, setting -t 2 -i 8,0. FIO has to be launched once perfstat reaches the 'sleeping' status in Iteration 1. Meaning sync step d) with perfstat. 

## Description of the workload types<br/>

w1) OLTP : 8k random rx/wr, 80% rx - 20% write<br/>
w2) Analytics: 16k andom rx/wr, 50% rx - 50% write<br/>
w3) Sequential read: 64k sequential read<br/>
w4) Sequential write: 64k sequential write<br/>
w5) 4k random reads (optional)<br/>

## Preparing the Clients

Assuming we're using Linux clients, you will need some packages on top of FIO. If you're using AWS you can run the script https://github.com/MichelePardini/fio_test_cvo/blob/master/aws_user_data_script.txt in the User Data when deploying the EC2 instances. This script will run some automatic updates plus cloning FIO from github. Besides these tasks, some manual steps are also required.
I always loging as root on these clients

1) For FIO (assuming /fio as install dir)<br/>
  > cd /fio<br/>
  > ./configure<br/>
  > make<br/>
  > make install<br/>
  
  fio will be located in /usr/local/bin

2) To have FIO in PATH, this is only for AWS version of CentOS, for other you can use the standard procedure
  > sudo su<br/>
  > vi /root/.bashrc<br/>
  add PATH=$PATH:/usr/local/bin<br/>
  save and exit<br/>
  > export PATH<br/>







