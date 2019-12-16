# fio_test_cvo

## This REPO contains a set of .fio configuration files that can be used to run performance test on a NetApp Cloud Volumes ONTAP.

This are the pre-req when configuring the testbed.
1) When you deploy a CVO from Cloud Manager, the configuration should be
  - 2 data aggregates, each composed of 6 EBS disk of 4TB each. Total of 12 disks that should allow 3GB/s max tput, or 144K op/s at 16Kb
  - Each aggregate cshould ontains 2 volumes of 2TB with Storage Efficiencies disabled. So 4 volumes in total. 
  - 8 clients running RHEL 7.7. 2 RHEL per volume, mounting the export with default parameters. You can use CenOS as well. 
  - The Working Set Size (WSS) varies depending on the instance type, the main rules are
      - Twice the size of the Memory (RAM) if the instance does not have an NVME FlashCache
      - Twice the size of the FlashCache, if present
      - The resulting WSS will be divided /# of volumes (so /4 in case you have 4 volumes, /2 if 2 volumes etc)
      
2) Use FIO v3.16 min.

<b>NOTE</b>: the configuration presented here is the best practice for large and very large instances, with 36 and more CPUs. If you're testing a standard type of instance (16 CPU) 2 volumes and 2 clients could be enough. 

The number you chose to use is very important as it'll be used to define some key paremeters in the fio configuration files.

I would recommend to use always - at least - 2 clients and 2 volumes. 1 client mapped to 1 volume via the desired protocol.

## How does FIO add workload with these configuration files?

I chose a progressive approach, instead of running at max from the beginning. So each client will start with 1 job composed of 2 threads (numjobs). After 2 minutes another job will be added, same with 2 threads. And so on until it reaches 8 jobs, for a total of 16 threads, per Linux client. And so if you have 2 Linux Client FIO will have 32 active jobs at the end.

It is paramount to get a perfstat during the whole test, that will run for ~27 minutes, to understand when the max/best performance was achieved, meaning how many jobs where running. 


## How to use it

a) Decide the number of volumes/clients and calculate the size of the WSS based on the instance type<br/>
b) Edit the 'create_dataset.fio' and change the params 'size' and 'nrfiles' to accomodate the new WSS. Avoid changing 'numjobs'<br/>
c) Run 'fio create_dataset.fio' from all clients to create the dataset on the 4 volumes<br/>
d) Once the dataset has been created, decide which workload to run. For seq rx you can run 'fio 64k_seq_wr.fio' from all the clients at the same time<br/>

## Use Perfstat

When running the test you should also capture a perfstat, setting -t 2 -i 8,0. FIO has to be launched once perfstat reaches the 'sleeping' status in Iteration 1. Meaning sync step d) with perfstat. 

## Example on how to edit the configuration files depending on the test scenario 

Say you want to test OLTP workload and your goal is 40.000 op/s on a c5.9xlarge. So you have
- 2 aggregates with 4 volumes in total
- 4 clients CentOS from AWS
- The WSS would be 72GB * 2 = 144Gb (you can then approx to 160Gb)

So, 160Gb /4 volumes = 40Gb. You will have to edit the 'create_dataset.fio' to create 40Gb using two params

> nrfiles=50<br/>
> size=50g<br/>

You can see that the default here is 50g, so the dataset (per volume) will be indeed 50g <b>per job</b>. Since we also have

> numjobs=2

Then the toal will be 50g * 2 = 100g, per volume (100 files of 1gb each). So, to have 40Gb in total, we have to change

> nrfiles=20<br/>
> size=20g<br/>

You can also decide not to change 'nrfiles', I like to have files of 1Gb. Don't change 'numjobs'. Save the changes and run fio against you 4 volumes to create 40*4=160Gb of WSS

The other important param that you may change is #iodepth. This represent the # of commands keept inflight agains the CVO by any job. You can see that the defaul is 

> iodepth=16

It could be a good place to start, increasing it could increase the tput but also the response time. So you want to find a sweet spot. How? At the beginning We said that the target of the PoC was 40.000 op/s. So the math would be

> iodepth = (target_IO * (latency/1000)) / (numbjobs * *#_Job * Linux_Clients)

How do you know the latency? Run a few ping from all clients. Say you TTL is 0.5ms. The result will be

(40.000 * (0.2 / 1000)) / (2 * 8 * 2) -> 20 / 32 = 0.6. 

Since iodepth can only be an integer you can set it to 1. 

> iodepth=1

If you're not looking for a specific target_IO but you want to max out the instance, then leave 16 and assess.


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







