# Bluefiled(BF) on Linux

In this document, we explain how to install Linux distributions on Bluefield. In our machine, we have the host CPU and the Arm of BF. We need to install the software on both the host and the BF side. 

Firstly, we will install the OFED and rshim drivers in the host. [Mellanox OFED - is a Mellanox tested and packaged version of OFED and supports two interconnect types using the same RDMA (remote DMA) and kernel bypass APIs called OFED verbs – InfiniBand and Ethernet.] 
Secondly, we will install OFED + Ubuntu in the BF. 
Thirdly, we will set the configuration of the BF (Separated mode vs Embedded mode).
Finally, we will test our OFED with one RDMA example and another with OFED examples.

Notes: 

1) In order to configure two remote machine with rdma, you need to have different ips for the interface tmfifo_net and two outgoing network interfacs. 
For example use for one machine, tmfifo_net0 192.168.100.1/24 enp133s0f0 192.168.0.20/24 enp133s0f1 192.168.0.21/24 
and for the another tmfifo_net0 192.169.100.1/24 ens2f0 192.169.0.20/24 ens2f1 192.169.0.21/24 
2) if the two outgoing network interfacs are not UP (after the configuration) check that your cables are connected correctly to the switch.

# Bluefiled on Ubuntu 18.04
## Requirment 
1) Host system is Ubuntu 18.04
2) BlueField NIC inserted in a host PCIe slot
3) USB cable connecting the NIC card and the host

## preinstall

Go to:
https://www.mellanox.com/products/software/bluefield
* Download (to host machine) MLNX_OFED. <mlnx_ofed>

![2](https://user-images.githubusercontent.com/28096724/87292707-e2487b80-c509-11ea-9263-7e3f5e8b48a1.png)

* Download BlueField BlueOS. <BlueOS_dir>
* Download (to host machine) BlueField Ubuntu Server 18.04 image. <bf_img.bfb>

![1](https://user-images.githubusercontent.com/28096724/87292641-c6dd7080-c509-11ea-8031-f21ee3f094da.png)


Run:

	sudo apt install linux-signed-generic-hwe-18.04
	sudo reboot --force
	sudo apt install build-essential debhelper autotools-dev dkms
	
## In the Host:

### OFED: 
	mount <mlnx_ofed>.iso /mnt
	sudo /mnt/uninstall --force
	sudo /mnt/mlnxofedinstall --add-kernel-support 
You should see that it is installed: 
Querying Mellanox devices firmware ...


### rshim drivers: 
	cd <BlueOS_dir>/src/drivers/rshim
	sudo modprobe -vr rshim_pcie
	sudo modprobe -vr rshim_net
	sudo modprobe -vr rshim_usb

##### First installation option: 

	dpkg-buildpackage -us -uc -nc
	sudo dpkg -i ../rshim-dkms_*.deb
	sudo modprobe rshim_usb
	sudo modprobe rshim_net
	
##### If the "First" did not work, pease  install it manually by:

	make -C /lib/modules/`uname -r`/build M=$PWD
	sudo modprobe rshim_usb
	sudo modprobe rshim_net
	
	vim /etc/udev/rules.d/91-tmfifo_net.rules
Add this line to tmfifo_net.rules:

SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="00:1a:ca:ff:ff:02", ATTR{type}=="1", NAME="tmfifo_net0", RUN+="/usr/sbin/ifup tmfifo_net0"
	
	reboot --force
	ip add
	
You should be able to see interface tmfifo_net and two outgoing network interfacs UP.
	
	sudo ifconfig tmfifo_net0 192.168.100.1/24 up
	sudo ifconfig <first_interface> 192.168.0.20 up
	sudo ifconfig <second_interface> 192.168.0.21 up
	
![12](https://user-images.githubusercontent.com/28096724/87291069-80871200-c507-11ea-8289-3cb2709d9a44.png)



## In the BF:

### Install the image to Bluefiled (Ubuntu 18.04 and OFED):
     cat  <bf_img.bfb> > /dev/rshim0/boot
     
Username: ssh ubuntu@192.168.100.2

Password: ubuntu


## Configuration:
The firmware default mode is Embedded, in order to use RoCE you should switch to Separated mode.
More information:
https://community.mellanox.com/s/article/BlueField-SmartNIC-Modes

In your host machine:

	mst start 
	mlxconfig -d /dev/mst/mt41682_pciconf0 s INTERNAL_CPU_MODEL=0
	reboot --force
	mst start 
	mlxconfig -d /dev/mst/mt41682_pciconf0 q | grep -i model
	
## Tests: (In the same machine)
	show_gids mlx5_1
use this index GID (RoCE doesn't work with LID-based routing), by picking any index from the GID table (it should probably be the same GID on both sides).
My GID index is 3. Change it in the RDMA example in both client and server.


	cat /sys/class/infiniband/mlx5_0/ports/1/gid_attrs/ndevs/
https://community.mellanox.com/s/article/howto-configure-roce-on-connectx-4

### Try the RDMA example:
	git clone https://github.com/LinaMaudlej/BF-linux.git
	cd server_rdma
	mkdir 
	./server 
	cd clinet
	mkdir 
	./client_rdma <port number>
	
![13](https://user-images.githubusercontent.com/28096724/87291927-be386a80-c508-11ea-9da7-b78d66f2a374.png)

### 1) In the same machine: Test by ibv_rc_pingpong and ibv_read_lat:
--ibv_rc_pingpong
	
	ibv_rc_pingpong -d mlx5_1 -g <gid_indx> 
	ibv_rc_ping_pong -g <gid_index> 192.169.0.21
	
-- ibv_read_lat

	ibv_read_lat -a 
	ibv_read_lat localhost -a 
	
### 2) In remote machines (two machines, let's assume server machine has tmfifo_net0 192.169.100.1/24 and client machine tmfifo_net0 192.168.100.1/24): 
-- ib_read_bw, in server side run:

	ib_read_bw 
in client side run:

	ib_read_bw 192.169.100.1

# Bluefiled on Centos 
TBD

# References 
https://drive.google.com/open?id=1IHpo1s06yhV-4PouQiFSYelbkUWEgCpa
https://docs.mellanox.com/display/BlueFieldSWv25011176/Installing+Popular+Linux+Distributions+on+BlueField

