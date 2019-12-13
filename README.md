# fio_test_cvo

This REPO contains a set of .fio configuration files that can be used to run performance test on a NetApp Cloud Volumes ONTAP.

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

When running the test you can also capture a perfstat, setting -t 2 -i 8,0. FIO has to be launched once perfstat reaches the 'sleeping' status in Iteration 1.
