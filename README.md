# Update Nvidia Linux driver on Debian 10.13 with 5.10.0-0.deb10.16-amd64 kernel for use with H200

Steps to update the Nvidia Linux driver from 535.86.10 to 550.127.08 to support H200 GPUs on AzHPC Debian 10.13 test image and steps to update OFED driver from MLNX_OFED_LINUX-23.10-0.5.5.0 to MLNX_OFED_LINUX-23.10-1.1.9.0.

Updates on production image should verify checksums of installed packages and perform extensive testing.  Skipped here for brevity.

## Update Nvidia Linux driver

### Remove existing Nvidia Linux driver and fabricmanager

Stop systemd services:

```bash
sudo systemctl stop nvidia-dcgm
sudo systemctl stop nvidia-fabricmanager
```

Uninstall driver:

```bash
sudo /usr/bin/nvidia-uninstall
$ cat /var/log/nvidia-uninstall.log
nvidia-installer log file '/var/log/nvidia-uninstall.log'
creation time: Fri Jan 17 01:16:17 2025
installer version: 535.86.10

PATH: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

nvidia-installer command line:
    /usr/bin/nvidia-uninstall

Using: nvidia-installer ncurses v6 user interface
-> Detected 96 CPUs online; setting concurrency level to 32.
-> If you plan to no longer use the NVIDIA driver, you should make sure that no X screens are configured to use the NVIDIA X driver in your X configuration file. If you used nvidia-xconfig to configure X, it may have created a backup of your original configuration. Would you like to run `nvidia-xconfig --restore-original-backup` to attempt restoration of the original X configuration file? (Answer: No)
-> Parsing log file:
-> done.
-> Validating previous installation:
-> done.
-> Uninstalling NVIDIA Accelerated Graphics Driver for Linux-x86_64 (1.0-5358610 (535.86.10)):
-> DKMS module detected; removing...
-> Unable to remove installed file '/lib/modules/5.10.0-0.deb10.16-amd64/kernel/drivers/video/nvidia.ko' (No such file or directory).
-> Unable to remove installed file '/lib/modules/5.10.0-0.deb10.16-amd64/kernel/drivers/video/nvidia-uvm.ko' (No such file or directory).
-> Unable to remove installed file '/lib/modules/5.10.0-0.deb10.16-amd64/kernel/drivers/video/nvidia-modeset.ko' (No such file or directory).
-> Unable to remove installed file '/lib/modules/5.10.0-0.deb10.16-amd64/kernel/drivers/video/nvidia-drm.ko' (No such file or directory).
-> Unable to remove installed file '/lib/modules/5.10.0-0.deb10.16-amd64/kernel/drivers/video/nvidia-peermem.ko' (No such file or directory).
WARNING: Failed to remove some installed files/symlinks. See /var/log/nvidia-uninstall.log for details
-> Failed to delete the directory '/etc/OpenCL/vendors' (Directory not empty).
-> Failed to delete the directory '/usr/share/nvidia' (Directory not empty).
-> Failed to delete the directory '/etc/OpenCL' (Directory not empty).
WARNING: Failed to delete some directories. See /var/log/nvidia-uninstall.log for details.
-> Unable to delete directories created by previous installation.
-> done.
-> Running depmod and ldconfig:
-> done.
-> Running `/usr/bin/systemctl daemon-reload`:
-> done.
-> Uninstallation of existing driver: NVIDIA Accelerated Graphics Driver for Linux-x86_64 (535.86.10) is complete.
```

Remove fabricmanager package:

```bash
sudo dpkg -r nvidia-fabricmanager-535
```

Reboot VM because driver may still be loaded.

```bash
sudo reboot
```

### Install newer Nvidia Linux driver

Download driver and install.

```bash
wget https://us.download.nvidia.com/tesla/550.127.08/NVIDIA-Linux-x86_64-550.127.08.run
sudo sh NVIDIA-Linux-x86_64-550.127.08.run --silent --dkms
```

Download and install matching fabricmanager.

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/debian10/x86_64/nvidia-fabricmanager-550_550.127.08-1_amd64.deb
sudo dpkg -i nvidia-fabricmanager-550_550.127.08-1_amd64.deb
```

Download and install libnvidia-nscq if it is not installed.

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/debian10/x86_64/libnvidia-nscq-550_550.127.08-1_amd64.deb
sudo dpkg -i libnvidia-nscq-550_550.127.08-1_amd64.deb
```

Download and install dcgm if it is not installed.

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/debian11/x86_64/datacenter-gpu-manager_3.1.8_amd64.deb
sudo dpkg -i ./datacenter-gpu-manager_3.1.8_amd64.deb
```

Reload systemd services

```bash
sudo systemctl daemon-reload
sudo systemctl restart nvidia-dcgm
sudo systemctl restart nvidia-fabricmanager
```

### Test

1. Test `nvidia-smi`.  Ensure that driver version 550.127.08 is in the output.

```bash
$ nvidia-smi
Fri Jan 17 01:28:11 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.127.08             Driver Version: 550.127.08     CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA H200                    Off |   00000001:00:00.0 Off |                    0 |
| N/A   30C    P0             78W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA H200                    Off |   00000002:00:00.0 Off |                    0 |
| N/A   29C    P0             76W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA H200                    Off |   00000003:00:00.0 Off |                    0 |
| N/A   28C    P0             76W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA H200                    Off |   00000008:00:00.0 Off |                    0 |
| N/A   30C    P0             78W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   4  NVIDIA H200                    Off |   00000009:00:00.0 Off |                    0 |
| N/A   28C    P0             76W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   5  NVIDIA H200                    Off |   0000000A:00:00.0 Off |                    0 |
| N/A   30C    P0             80W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   6  NVIDIA H200                    Off |   0000000B:00:00.0 Off |                    0 |
| N/A   29C    P0             77W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   7  NVIDIA H200                    Off |   0000000C:00:00.0 Off |                    0 |
| N/A   30C    P0             76W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

2. Test `dcgmi -r 1`.  It should finish in a few seconds.  If it does not, it indicates that something is wrong with the installation.

```bash
$ dcgmi diag -r 1
Successfully ran diagnostic for group.
+---------------------------+------------------------------------------------+
| Diagnostic                | Result                                         |
+===========================+================================================+
|-----  Metadata  ----------+------------------------------------------------|
| DCGM Version              | 3.1.8                                          |
| Driver Version Detected   | 550.127.08                                     |
| GPU Device IDs Detected   | 2335,2335,2335,2335,2335,2335,2335,2335        |
|-----  Deployment  --------+------------------------------------------------|
| Denylist                  | Pass                                           |
| NVML Library              | Pass                                           |
| CUDA Main Library         | Pass                                           |
| Permissions and OS Blocks | Pass                                           |
| Persistence Mode          | Fail                                           |
| Error                     | Persistence mode for GPU 0 is currently disab  |
|                           | led. The DCGM diagnostic requires peristence   |
|                           | mode to be enabled. Enable persistence mode b  |
|                           | y running "nvidia-smi -i <gpuId> -pm 1 " as r  |
|                           | oot., Persistence mode for GPU 1 is currently  |
|                           |  disabled. The DCGM diagnostic requires peris  |
|                           | tence mode to be enabled. Enable persistence   |
|                           | mode by running "nvidia-smi -i <gpuId> -pm 1   |
|                           | " as root., Persistence mode for GPU 2 is cur  |
|                           | rently disabled. The DCGM diagnostic requires  |
|                           |  peristence mode to be enabled. Enable persis  |
|                           | tence mode by running "nvidia-smi -i <gpuId>   |
|                           | -pm 1 " as root., Persistence mode for GPU 3   |
|                           | is currently disabled. The DCGM diagnostic re  |
|                           | quires peristence mode to be enabled. Enable   |
|                           | persistence mode by running "nvidia-smi -i <g  |
|                           | puId> -pm 1 " as root., Persistence mode for   |
|                           | GPU 4 is currently disabled. The DCGM diagnos  |
|                           | tic requires peristence mode to be enabled. E  |
|                           | nable persistence mode by running "nvidia-smi  |
|                           |  -i <gpuId> -pm 1 " as root., Persistence mod  |
|                           | e for GPU 5 is currently disabled. The DCGM d  |
|                           | iagnostic requires peristence mod              |
| Environment Variables     | Pass                                           |
| Page Retirement/Row Remap | Pass                                           |
| Graphics Processes        | Pass                                           |
| Inforom                   | Pass                                           |
+---------------------------+------------------------------------------------+
```

Run NCCL all-reduce as real test:

```bash
module load mpi/hpcx

mpirun --allow-run-as-root \
    -np 8 \
    --bind-to numa \
    --map-by ppr:8:node \
    -x LD_LIBRARY_PATH=/usr/local/nccl-rdma-sharp-plugins/lib:$LD_LIBRARY_PATH \
    -mca coll_hcoll_enable 0 \
    -x NCCL_IB_PCI_RELAXED_ORDERING=1 \
    -x UCX_TLS=tcp \
    -x UCX_NET_DEVICES=eth0 \
    -x CUDA_DEVICE_ORDER=PCI_BUS_ID \
    -x NCCL_SOCKET_IFNAME=eth0 \
    -x NCCL_DEBUG=WARN \
    -x NCCL_TOPO_FILE=/opt/microsoft/ndv5-topo.xml \
    -x NCCL_MIN_NCHANNELS=32 \
    /opt/nccl-tests/build/all_reduce_perf -b 8G -e 8G -f 2 -g 1 -c 1 -n 10
# nThread 1 nGpus 1 minBytes 8589934592 maxBytes 8589934592 step: 2(factor) warmup iters: 5 iters: 10 agg iters: 1 validation: 1 graph: 0
#
# Using devices
#  Rank  0 Group  0 Pid  23690 on test-h200-deb device  0 [0x00] NVIDIA H200
#  Rank  1 Group  0 Pid  23691 on test-h200-deb device  1 [0x00] NVIDIA H200
#  Rank  2 Group  0 Pid  23692 on test-h200-deb device  2 [0x00] NVIDIA H200
#  Rank  3 Group  0 Pid  23693 on test-h200-deb device  3 [0x00] NVIDIA H200
#  Rank  4 Group  0 Pid  23694 on test-h200-deb device  4 [0x00] NVIDIA H200
#  Rank  5 Group  0 Pid  23695 on test-h200-deb device  5 [0x00] NVIDIA H200
#  Rank  6 Group  0 Pid  23696 on test-h200-deb device  6 [0x00] NVIDIA H200
#  Rank  7 Group  0 Pid  23697 on test-h200-deb device  7 [0x00] NVIDIA H200
NCCL version 2.19.3+cuda12.2
#
#                                                              out-of-place                       in-place
#       size         count      type   redop    root     time   algbw   busbw #wrong     time   algbw   busbw #wrong
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)            (us)  (GB/s)  (GB/s)
  8589934592    2147483648     float     sum      -1    31517  272.55  476.96      0    31516  272.56  476.98      0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 476.967
#
```

## Update OFED driver from MLNX_OFED_LINUX-23.10-0.5.5.0 to MLNX_OFED_LINUX-23.10-1.1.9.0

### Remove old driver (MLNX_OFED_LINUX-23.10-0.5.5.0)

```bash
sudo /etc/init.d/openibd stop
sudo ofed_uninstall.sh
```

### Install new driver (23.10-1.1.9.0)

Download and install new driver

```bash
wget https://content.mellanox.com/ofed/MLNX_OFED-23.10-1.1.9.0/MLNX_OFED_LINUX-23.10-1.1.9.0-debian10.13-x86_64.tgz
tar xzf MLNX_OFED_LINUX-23.10-1.1.9.0-debian10.13-x86_64.tgz
sudo ./MLNX_OFED_LINUX-23.10-1.1.9.0-debian10.13-x86_64/mlnxofedinstall --add-kernel-support  --skip-unsupported-devices-check --without-fw-update
sudo /etc/init.d/openibd restart
```

Check the new version is reported

```bash
ofed_info -s
MLNX_OFED_LINUX-23.10-1.1.9.0:
ibdev_info
hca_id:	mlx5_ib0
	transport:			InfiniBand (0)
	fw_ver:				28.40.1702
	node_guid:			0015:5dff:fe34:0033
	sys_image_guid:			0427:2803:0126:770c
	vendor_id:			0x02c9
	vendor_part_id:			4126
	hw_ver:				0x0
	board_id:			MSF0000000047
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			1
			port_lid:		1227
			port_lmc:		0x00
			link_layer:		InfiniBand

hca_id:	mlx5_ib1
	transport:			InfiniBand (0)
	fw_ver:				28.40.1702
	node_guid:			0015:5dff:fe34:0034
	sys_image_guid:			0427:2803:0226:770d
	vendor_id:			0x02c9
	vendor_part_id:			4126
	hw_ver:				0x0
	board_id:			MSF0000000047
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			1
			port_lid:		1304
			port_lmc:		0x00
			link_layer:		InfiniBand

hca_id:	mlx5_ib2
	transport:			InfiniBand (0)
	fw_ver:				28.40.1702
	node_guid:			0015:5dff:fe34:0035
	sys_image_guid:			0427:2803:0326:770e
	vendor_id:			0x02c9
	vendor_part_id:			4126
	hw_ver:				0x0
	board_id:			MSF0000000047
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			1
			port_lid:		1266
			port_lmc:		0x00
			link_layer:		InfiniBand

hca_id:	mlx5_ib3
	transport:			InfiniBand (0)
	fw_ver:				28.40.1702
	node_guid:			0015:5dff:fe34:0036
	sys_image_guid:			0427:2803:0426:770f
	vendor_id:			0x02c9
	vendor_part_id:			4126
	hw_ver:				0x0
	board_id:			MSF0000000047
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			1
			port_lid:		1366
			port_lmc:		0x00
			link_layer:		InfiniBand

hca_id:	mlx5_ib4
	transport:			InfiniBand (0)
	fw_ver:				28.40.1702
	node_guid:			0015:5dff:fe34:0037
	sys_image_guid:			0427:2803:0526:7710
	vendor_id:			0x02c9
	vendor_part_id:			4126
	hw_ver:				0x0
	board_id:			MSF0000000047
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			1
			port_lid:		1426
			port_lmc:		0x00
			link_layer:		InfiniBand

hca_id:	mlx5_ib5
	transport:			InfiniBand (0)
	fw_ver:				28.40.1702
	node_guid:			0015:5dff:fe34:0038
	sys_image_guid:			0427:2803:0626:7711
	vendor_id:			0x02c9
	vendor_part_id:			4126
	hw_ver:				0x0
	board_id:			MSF0000000047
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			1
			port_lid:		1476
			port_lmc:		0x00
			link_layer:		InfiniBand

hca_id:	mlx5_ib6
	transport:			InfiniBand (0)
	fw_ver:				28.40.1702
	node_guid:			0015:5dff:fe34:0039
	sys_image_guid:			0427:2803:0726:7712
	vendor_id:			0x02c9
	vendor_part_id:			4126
	hw_ver:				0x0
	board_id:			MSF0000000047
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			1
			port_lid:		1466
			port_lmc:		0x00
			link_layer:		InfiniBand

hca_id:	mlx5_ib7
	transport:			InfiniBand (0)
	fw_ver:				28.40.1702
	node_guid:			0015:5dff:fe34:003a
	sys_image_guid:			0427:2803:0826:7713
	vendor_id:			0x02c9
	vendor_part_id:			4126
	hw_ver:				0x0
	board_id:			MSF0000000047
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			1
			port_lid:		1554
			port_lmc:		0x00
			link_layer:		InfiniBand

hca_id:	mlx5_an0
	transport:			InfiniBand (0)
	fw_ver:				16.30.5000
	node_guid:			0022:48ff:fe65:bc8d
	sys_image_guid:			0000:0000:0000:0000
	vendor_id:			0x02c9
	vendor_part_id:			4122
	hw_ver:				0x80
	board_id:			MSF0000000041
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		1024 (3)
			sm_lid:			0
			port_lid:		0
			port_lmc:		0x00
			link_layer:		Ethernet
```