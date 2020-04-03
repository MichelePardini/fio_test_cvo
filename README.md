# fio_test_cvo

## This REPO contains a set of .fio configuration files that can be used to run performance test on a NetApp Cloud Volumes ONTAP.

## This are the pre-req when configuring the testbed.

When you deploy a CVO from Cloud Manager, the configuration should be
  - 1 data aggregate. In AWS it should be composed of 6 EBS disk of 6TB; in Azure 12 disks of 6TB.
  - If you are testing an instance with FlashCache then the tiering to object store *MUST* be disabled
  - Create 4 volumes, each one with size of 2TB. If you are testing an instance with FlashCache then you must create the volumes with       Storage Efficiencies disabled
  - This scenario assumes you use NFS, but also applies to SMB and iSCSI. When I say 'mount' I mean connect 1 client to 1 volume,           regardless of the protocol used

For the test
  - Use FIO v3.19 min. Man page here https://fio.readthedocs.io/en/latest/fio_doc.html
  - Create 4 clients running RHEL 7.7.Each RHEL will mount 1 volume. You can use CenOS as well. The client VM type depends on the test       you want to run. The clients should be deployed in the same subnet as the CVO. 
  - On each client, create a directory to be used as mount point and then mount the respective volume from the CVO (note that you should     use the same name for the mount point on all Linux Clients)
  - The Working Set Size (WSS) varies depending on the instance type, the main rules are
      - At least 10x the size of the Memory (RAM) if the instance does not have an NVME FlashCache
      - At least 2x the size of the NVMe FlashCache, if present
      - The resulting WSS will be split by the number of volumes

<b>NOTE</b>: the configuration presented here is the best practice for large and very large instances, with 36 and more CPUs. If you're testing a standard type of instance (16 CPU) 2 volumes and 2 clients could be enough. 

I would recommend to use always - at least - 2 clients and 2 volumes. 1 client mapped to 1 volume via the desired protocol, best is 4 volumes and 4 clients

## Preparing the Clients

First, you have to chose the instance type. Don't pick a small instance, it should be able to manage at least 1GB/s via network, have 8 CPU and 32Gb of memory.

Assuming we're using Linux clients, you will need some packages on top of FIO. If you're using AWS you can run the script https://github.com/MichelePardini/fio_test_cvo/blob/master/aws_user_data_script.txt in the User Data when deploying the EC2 instances. This script will run some automatic updates plus cloning FIO from github. Otherwise you can just open the txt file and run the commands manually. Besides these tasks, some manual steps are also required. I always loging as root

1) For FIO (assuming /fio as install dir)<br/>
  > cd /fio<br/>
  > ./configure<br/>
  > make<br/>
  > make install<br/>
  
  FIO will be located in /usr/local/bin

2) To have FIO in PATH, this is only for AWS version of CentOS, for other you can use the standard procedure
  > sudo su<br/>
  > vi /root/.bashrc<br/>
  add PATH=$PATH:/usr/local/bin<br/>
  save and exit<br/>
  > export PATH<br/>

## How does FIO add workload with these configuration files?

I've chosen a progressive approach, instead of running at max throttle from the beginning each client will start with 1 job composed of 2 threads (numjobs). After 2 minutes another job will be added, same with 2 threads. And so on until it reaches 8 jobs, for a total of 16 threads, per Linux client. And so if you have 2 Linux Client FIO will have 32 active jobs at the end. 

## How to use this setup

a) By now you should have decided the number of volumes/clients and calculated the size of the WSS based on the instance type. Each        client is mouting 1 volume<br/>
b) Edit the 'create_dataset.fio' and change the params 'size' and 'nrfiles' to accomodate the new WSS, then the 'directory' param to        match your mount point name. Avoid changing 'numjobs'<br/>
c) Run 'fio create_dataset.fio' to create the dataset on the volumes <br/>
d) Once the dataset has been created, decide which workload to run. For seq rx you can run 'fio 64k_seq_wr.fio' from all the clients at    the same time<br/>

## Using Perfstat/PerfArchive

For users that are familiar with NetApp and that want to investigate on CVO stats you can also get a perfstat during the whole test, that will run for ~28 minutes, to understand when the max/best performance was achieved, meaning how many jobs where running. 

When running the test you should also capture a perfstat, setting -t 2 -i 8,0. FIO has to be launched once perfstat reaches the 'sleeping' status in Iteration 1. Meaning sync step d) with perfstat. 

Otherwise you can trigger a PerfArchive once all the tests are completed. 

## Example on how to edit the configuration files depending on the test scenario 

Say you want to test OLTP workload and your goal is 40.000 op/s on a c5.9xlarge. So you have
- 1 aggregate with 4 volumes in total
- 4 clients CentOS from AWS
- The WSS would be 72GB * 10 = 720Gb (better then approx to 1TB)

So, 1TB /4 volumes = 250Gb per volume. You will have to edit the 'create_dataset.fio' to create 250Gb using two params

> nrfiles=50<br/>
> size=50g<br/>

You can see that the default here is 50g, so the dataset (per volume) will be indeed 50g <b>per job</b>. But because we also have

> numjobs=2

Then the total will be 50g * 2 = 100g, per volume. The param 'nrfile' defines the total number of files per job, so you have size=50g and nrfiles=50 then you have 50 files of 1gb each <b>per job</b>, 100 files of 1gb in total. 

Back to our example, to have 250Gb in total we have to change

> nrfiles=12<br/>
> size=125g<br/>

You can also decide not to change 'nrfiles'. Files with larger size will result in a faster WSS creation. Don't change 'numjobs'. Save the changes and run fio against your 4 volumes to create 250*4=1TBGb of WSS

The last parameter to change is 'directory' , you can see the default is

> directory=/dataset

This will need to be changed to match your mount point. Since you will use the same file on all clients it's better to create the same mount point name on all, as said before. Save the changes and copy the file on all clients in the /fio directory

## Creating the WSS

Login into each client and run the fio command (from /fio directory)

> ./fio create_dataset.fio

Once completed you will get the classic FIO report, but in this case it's not relevant, you can ignore it. You should have created the WSS on each volume. You can double check running 'ls' on all clients.

## Description of the workload types<br/>

w1) OLTP : 8k random rx/wr, 80% rx - 20% write<br/>
w2) Analytics: 16k random rx/wr, 50% rx - 50% write<br/>
w3) Sequential read: 100% 64k sequential read<br/>
w4) Sequential write: 100% 64k sequential write<br/>
w5) 4k random reads: 100% 4k random reads<br/>

## Modifying the FIO workload configuration file

First, you have to edit the fio workload confifuration file and change the 'directory' parameter to match your mount point name, like already done to create the WSS. Once done copy the file on all clients in the /fio directory.

> directory=/dataset

The other important param that you may change is 'iodepth'. This represent the # of commands kept inflight agains the CVO by any job. You can see that the defaul is 

> iodepth=16

It could be a good place to start, increasing it could increase the tput but also the response time. So you want to find a sweet spot. How? In the example mentioned above we said that the target op/s is 40.000 op/s. So the math would be

> iodepth = (target_IO * (latency/1000)) / (numbjobs * *#_Job * Linux_Clients)

How do you know the latency? Run a few ping from all clients. Say you TTL is 0.5ms. The result will be

> (40.000 * (0.2 / 1000)) / (2 * 8 * 2) -> 20 / 32 = 0.6. 

Since iodepth can only be an integer you can set it to 1. 

> iodepth=1

If you're not looking for a specific target_IO but you want to max out the instance, then leave 16 and assess.

## Running FIO workloads

You should have SSH open to all your Linux clients, then you can run - at the same time - the fio command for the desired workload, for instance if you want to run 4k random (from /fio directory)

> ./fio 4k_random.fio 

This will run for ~28 minutes. At the end of each run you will get a report on the console of your Clients. If you want to save this report either you can copy paste or you can run something like this

> ./fio 4k_random.fio > /tmp/4k_report_client1.log

So the report will be saved into a text file that you can check out later. In the report you will have the most important information about the client stats, such avg rx/wr latency and avg rx/wr TPUT. 

In this page https://tobert.github.io/post/2014-04-17-fio-output-explained.html you will find all the info on how to read the FIO output

## Running customized workload

The configuration files I have here are based on most commong workloads we see at customers, but you can run whatever type of workload you want. You just need to edit the config file and modify the key parameters in the [global] section. 

Here https://github.com/axboe/fio/tree/master/examples you can find lots of examples. 


