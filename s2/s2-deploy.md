# s2 simple deployment guide

## Requirement

You need at least 3 servers to deploy a storage service.

Each of them must:
-   have at least 4GB memory.
-   run Centos 7.0+ Operating System

Host address example:
-   172.18.2.2
-   172.18.2.3
-   172.18.2.4

## Installation

#### Each machine needs to complete the following operations.

1、Disable firewalld and selinux. 

/etc/selinux/config, change the line to be.

```
SELINUX=disabled
```
Run command to disable firewalld.
    
```sh
$ systemctl disable firewalld

$ reboot
```
2、Enable sshd on 10022 port.

/etc/ssh/sshd_config, change the line to be.
    
```
Port 10022
```
    
Run command to restart sshd.

```sh
$ systemctl restart sshd
```

#### Choose one of the servers as the controlling host, where you do all the following steps.

1、Install git package.

```sh
$ yum install git
```

2、Generate the ssh key pairs and add the public key to your git account to avoid enter the username and password many 
times during the installation. [See this](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh).

3、Clone the `s2-init` repo. You should not be asked to enter username and password if you complete the last step successfully.

```sh
$ git clone git@github.com:bsc-s2/s2-init.git
```

Confirm `github.com` address host can ping.

4、Install packages for controlling host (any host that has ssh access (with password) to all others in the cluster).
Don't continue until you see the output `OK: main`.

```sh
$ cd s2-init/
$ sh init-ansible-server.sh
```

The script execute `clone repositories` will cost a lot of time, you can manually download the repo to the local host，
and upload files to `s2-init/source-code/` folder. repos list check `init-project-code.sh`.

```sh
$ cat init-project-code.sh
```

Do step 4 again.

5、Create inventory files to describe the cluster in the inventories folder.

Copy a sample inventory folder (`./inventories/<sample_folder>`) to `<your_inventory_folder>`.

```sh
$ cd inventories/
$ cp -rf baishan-2copy/ s2-cluster
```

Update the cluster name with the name of your inventory folder in `<your_inventory_folder>/group_vars/all/conf.yaml`.

```sh
$ vim s2-cluster/group_vars/all/conf.yaml

---
cluster         : 's2-cc'
conf:
  ......
  
  group:
      space_capacity: "{{ 100 * 1024 * 1024 * 1024 }}"
      num_capacity:   "{{ 50 * 1024 }}"
      tactics:

        - idc: .s2.cc
          max_active: 50
          min_active: 40
          level: 'optimal'
          active:
            - ['ATA', '.s2.cc',       ]
            - ['ATA', '.s2.cc',       ]
          std:
            - ['ATA', '.s2.cc',       ]
            - ['ATA', '.s2.cc',       ]
            
   ......
```

Set our hosts in `<inventory_folder>/hosts.yaml` to use dynamic inventory.

Here is the sample below.

```sh
$ vim s2-cluster/hosts.yaml

---
idc-s2-cc:
    vars:
        idc: .s2.cc
        idc_type: center

    hosts:
        - 172.18.2.2
        - 172.18.2.3
        - 172.18.2.4
s2:
    children:
        - idc-s2-cc

zookeeper:
    hosts:
        - [172.18.2.2,  { zookeeper_id: 1 }]
        - [172.18.2.3,  { zookeeper_id: 2 }]
        - [172.18.2.4,  { zookeeper_id: 3 }]
kafka-cluster-1:
    hosts:
        172.18.2.2: { zookeeper_id: 1, broker_id: 0 }
        172.18.2.3: { zookeeper_id: 2, broker_id: 1 }
        172.18.2.4: { zookeeper_id: 3, broker_id: 2 }
kafka:
    children:
        - kafka-cluster-1

etcd:
    hosts:
        - 172.18.2.2
        - 172.18.2.3
        - 172.18.2.4
```

Configure `<inventory_folder>/indexed_hosts.yaml`.

```sh
$ vim s2-cluster/indexed_hosts.yaml

indexed_hosts:
  172.18.2.2: 1
  172.18.2.3: 2
  172.18.2.4: 3
```

Config mysql group in `<inventory_folder>/group_vars/all/mysql_replication.yaml`.

```sh
$ vim s2-cluster/group_vars/all/mysql_replication.yaml

---

mysql_db_user: s2-cluster

#   1<--------------->3
#   ^                 ^
#   |                 |
#   |                 |
#   |                 |
#   5<--------------->7

mysql_default_topology:
  backup: 2
  src:
    "1": ["1", "3",    ]
    "3": ["1",      "5"]
    "5": [     "3", "5"]

  idc:
    172.18.2.2: .s2.cc           # DB-D480G-M128G
    172.18.2.3: .s2.cc           # DB-D480G-M128G
    172.18.2.4: .s2.cc           # DB-D480G-M128G
    
mysql_backup_hosts:
  .s2.cc: "ss.bscstorage.com"


mysql_replication:
  "3401": { 172.18.2.2: 1, 172.18.2.3: 3, 172.18.2.4: 5}
  "3402": { 172.18.2.2: 1, 172.18.2.3: 3, 172.18.2.4: 5}
  "3403": { 172.18.2.2: 1, 172.18.2.3: 3, 172.18.2.4: 5}

  "3501": { 172.18.2.2: 1, 172.18.2.3: 3, 172.18.2.4: 5}
  "3502": { 172.18.2.2: 1, 172.18.2.3: 3, 172.18.2.4: 5}
  "3503": { 172.18.2.2: 1, 172.18.2.3: 3, 172.18.2.4: 5}
```

Update the ports in `<inventory_folder>/group_vars/all/table.yaml`.

```sh
$ vim s2-cluster/group_vars/all/table.yaml

---
tables:
    key:
        - { from: [ '1000000000000000000', '',  "" ], db: "3401" }

    phy:
        - { from: [ "00000000" ], db: "3501" }
        - { from: [ "01000000" ], db: "3501" }
        - { from: [ "02000000" ], db: "3501" }
        - { from: [ "03000000" ], db: "3501" }
        - { from: [ "04000000" ], db: "3501" }
        - { from: [ "05000000" ], db: "3501" }
        - { from: [ "06000000" ], db: "3501" }
        - { from: [ "07000000" ], db: "3501" }
        - { from: [ "08000000" ], db: "3501" }
        - { from: [ "09000000" ], db: "3501" }
        - { from: [ "0a000000" ], db: "3501" }

    errlog:
        - { from: [],  db: "3402" }
    cluster_log:
        - { from: [],  db: "3402" }
    group:
        - { from: [],  db: "3402" }
    ec:
        - { from: [],  db: "3402" }
    group_partition:
        - { from: [],  db: "3402" }
    move_task:
        - { from: [],  db: "3402" }
    bucket:
        - { from: [],  db: "3402" }
    user:
        - { from: [],  db: "3402" }
    admin_user:
        - { from: [],  db: "3402" }
    bound_user:
        - { from: [],  db: "3402" }
    accesskey:
        - { from: [],  db: "3402" }
    group_move_task:
        - { from: [],  db: "3402" }
    arc_move_task:
        - { from: [],  db: "3402" }
    throttle:
        - { from: [],  db: "3402" }

    statistics_5min:
        - { from: [],  db: "3502" }
    statistics_day:
        - { from: [],  db: "3502" }
    statistics_filetype_5min:
        - { from: [],  db: "3502" }
    statistics_filetype_day:
        - { from: [],  db: "3502" }
    bill_daily:
        - { from: [],  db: "3502" }
    bill_bucket_total:
        - { from: [],  db: "3502" }
    bill_storage_minutely:
        - { from: [],  db: "3502" }
    bill_storage_hourly:
        - { from: [],  db: "3502" }
    bill_storage_daily:
        - { from: [],  db: "3502" }
    bill_storage_total:
        - { from: [],  db: "3502" }
    bill_video:
        - { from: [],  db: "3502" }
    bill_image:
        - { from: [],  db: "3502" }

    # ec_statistics:
    #     - { from: [],  db: "3402" }

    pipeline_queue:
        - { from: [],  db: "3502"}
    pipeline_task:
        - { from: [ '1000000000000000000' ], db: "3502" }
        - { from: [ '1100000000000000000' ], db: "3502" }

    transcoder_preset:
        - { from: [], db: "3502"}
```


Set stage to test in `<inventory_folder>/group_vars/all/stage.yaml` if you are setting up a test env.

```sh
$ vim s2-cluster/group_vars/all/stage.yaml

---
stage: test
```

6、Use inventory.py.

When you run ansible, you must specify a inventory host file: `./inventory/<inventory_folder>/inventory.py`.<br>
When absent, use env-var ANSIBLE_INVENTORY:

```sh
$ export ANSIBLE_INVENTORY=./inventories/s2-cluster/inventory.py.
```

You need to copy inventory.py from baishan-3copy/inventory.py to your own inventory directory.

You can enter your inventory directory, run python inventory.py --list --readable, to print configure list.

```sh
$ python inventories/s2-cluster/inventory.py --list --readable
```

7、Add ssh public key of the controlling host to all hosts in cluster.

Init ssh key ( Input root password If asked ).

```sh
$ sh init.sh -i inventories/s2-cluster/inventory.py -c key
```

When connecting to other hosts, you are no longer prompted to enter a password, which means that the above script is executed successfully.


8、Init system environment (need enter password to decrypt etcd/zookeeper account).

Entry the password to continue.

```sh
$ sh init.sh -i inventories/s2-cluster/inventory.py -b playbooks/init-system-env-common.yaml
```

9、Install s2.

Install everything and make sure no failure happens (failed: 0 for every host).

Entry the password to continue.

```sh
$ sh init.sh -i inventories/s2-cluster/inventory.py -b playbooks/install-s2.yaml
```

This operation tasks a long time.

About mounting block devices:

Two 5.0G capacity files will be created and they will be attached to `/dev/loop[x]` as pseudo drive. Then they will be mounted to `/s2/drive/001` and `/s2/drive/002`.

For production env, all drives other than the /dev/sda will be formatted as xfs and will be mounted to /s2/drive/001.

```sh
$ df -hT
文件系统       类型      容量  已用  可用 已用% 挂载点
/dev/vda1      xfs        40G   19G   22G   48% /
devtmpfs       devtmpfs  1.9G     0  1.9G    0% /dev
tmpfs          tmpfs     1.9G  300K  1.9G    1% /dev/shm
tmpfs          tmpfs     1.9G  185M  1.7G   10% /run
tmpfs          tmpfs     1.9G     0  1.9G    0% /sys/fs/cgroup
tmpfs          tmpfs     380M     0  380M    0% /run/user/0
/dev/loop1     xfs       5.0G   34M  5.0G    1% /s2/drive/001
/dev/loop2     xfs       5.0G   34M  5.0G    1% /s2/drive/002
```

10、Configure s2.

Run command `s2mgr.py` to entry management terminal.

```sh
$ s2mgr.py
```

List nodes and set node role.

```sh
$ node l
$ node role set Storage
Change roles on 3 nodes? :y
```

Add partitions (groups will be created in a few minutes).

```sh
$ partition add --free 1g --size 1g -y
```

Wait a few minutes and check the partition.

```shell
$ partition ll
$ group lsactive
```

Now, complete the installation of the service.


## Test

Register accounts to the database.

```sh
$ python /usr/local/s2/current/test/test_account.py
```

To run tests.

```sh
$ python /usr/local/s2/current/test/run_all_fronttests.py -v
```


