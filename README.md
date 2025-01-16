# Update Nvidia Linux driver on Debian 10.13 with 5.10.0-0.deb10.16-amd64 kernel

Steps used to update the Nvidia Linux driver from 535.86.10 to 565.57.01 to support H200 GPUs on Debian 10.13 test image.  Also includes steps to update OFED driver from MLNX_OFED_LINUX-23.10-0.5.5.0 to MLNX_OFED_LINUX-23.10-1.1.9.0.

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
```

Remove fabricmanager package:

```bash
sudo dpkg -r nvidia-fabricmanager-535
```

### Install newer Nvidia Linux driver

Download driver and install.

```bash
wget https://us.download.nvidia.com/tesla/565.57.01/NVIDIA-Linux-x86_64-565.57.01.run
sudo sh NVIDIA-Linux-x86_64-565.57.01.run --silent --dkms
```

Download and install matching fabricmanager version.  There is no supported version for Debian 10, so I tried the only version available Debian 12 and it worked.

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/nvidia-fabricmanager-565_565.57.01-1_amd64.deb .
sudo dpkg -i nvidia-fabricmanager-565_565.57.01-1_amd64.deb
```

Reload systemd services

```bash
sudo systemctl daemon-reload
sudo systemctl restart nvidia-dcgm
sudo systemctl restart nvidia-fabricmanager
```

### Test

Make sure `nvidia-smi` works:

```bash
$ nvidia-smi
Wed Jan 15 23:35:00 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 565.57.01              Driver Version: 565.57.01      CUDA Version: 12.7     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA H200                    Off |   00000001:00:00.0 Off |                    0 |
| N/A   34C    P0            123W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA H200                    Off |   00000002:00:00.0 Off |                    0 |
| N/A   33C    P0            123W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA H200                    Off |   00000003:00:00.0 Off |                    0 |
| N/A   31C    P0            121W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA H200                    Off |   00000008:00:00.0 Off |                    0 |
| N/A   33C    P0            119W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   4  NVIDIA H200                    Off |   00000009:00:00.0 Off |                    0 |
| N/A   31C    P0            121W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   5  NVIDIA H200                    Off |   0000000A:00:00.0 Off |                    0 |
| N/A   35C    P0            119W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   6  NVIDIA H200                    Off |   0000000B:00:00.0 Off |                    0 |
| N/A   32C    P0            120W /  700W |       1MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   7  NVIDIA H200                    Off |   0000000C:00:00.0 Off |                    0 |
| N/A   34C    P0            120W /  700W |       1MiB / 143771MiB |      0%      Default |
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
#  Rank  0 Group  0 Pid  39280 on test-h200-dsvm device  0 [0x00] NVIDIA H200
#  Rank  1 Group  0 Pid  39281 on test-h200-dsvm device  1 [0x00] NVIDIA H200
#  Rank  2 Group  0 Pid  39282 on test-h200-dsvm device  2 [0x00] NVIDIA H200
#  Rank  3 Group  0 Pid  39283 on test-h200-dsvm device  3 [0x00] NVIDIA H200
#  Rank  4 Group  0 Pid  39284 on test-h200-dsvm device  4 [0x00] NVIDIA H200
#  Rank  5 Group  0 Pid  39285 on test-h200-dsvm device  5 [0x00] NVIDIA H200
#  Rank  6 Group  0 Pid  39286 on test-h200-dsvm device  6 [0x00] NVIDIA H200
#  Rank  7 Group  0 Pid  39287 on test-h200-dsvm device  7 [0x00] NVIDIA H200
NCCL version 2.19.3+cuda12.2
#
#                                                              out-of-place                       in-place
#       size         count      type   redop    root     time   algbw   busbw #wrong     time   algbw   busbw #wrong
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)            (us)  (GB/s)  (GB/s)
  8589934592    2147483648     float     sum      -1    31533  272.41  476.72      0    31441  273.20  478.11      0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 477.416
#
```

## Update OFED driver from MLNX_OFED_LINUX-23.10-0.5.5.0 to MLNX_OFED_LINUX-23.10-1.1.9.0

### Remove old driver ()

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