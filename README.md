# Set up

Follow the Helium user manual to set up connection between the host and the DPU.

Go through sections 6 to 8.

Differences to note:

1. Under section 6.2 Host-side Driver Dependencies, step 4, the warning: `PCIe ACS override enabled` is not seen. Can ignore.

2. Under section 7.1 Enable Virtual Port on the Host, step 1, note the difference in root PCI bus address. You should observe that our address is `[0000:30]-+-02.0-[31]--+-00.0`.

In step 2, on the DPU, do not include `2>/dev/null` to view the output. Before `rmmod mgmt_net`, run `ifconfig mvmgmt0 down`.

Change the parameters of `modprobe pcie_ep` to the following: `modprobe pcie_ep host_sid=0x33000 pem_num=0 epf_num=0`. Note that the `host_sid` parameter is changed to `0x33000` to match the root PCI bus address `0000:30`.

Follow through the rest of the steps. You should be able to see some RX and TX packets when running `ifconfig mvmgmt0`.

# Set up 100G ports

The DPU `eth1` interface is connected to the Poseidon `ens114f1np1` interface.

First, advertise the 100G link on DPU side with

```bash
ethtool -s eth0 advertise 0x8000000000
ethtool -s eth1 advertise 0x8000000000
```

Reboot the DPU. You should see as part of the boot-up
```bash
In:    serial
Out:   serial
Err:   serial
CGX0 LMAC0 [100G_R]
CGX2 LMAC0 [100G_R]
```

Ensure the link is up with
```bash
ip link set dev eth1 down
ip link set dev eth1 up
```

On the Poseidon side, enable autoneg.
```bash
sudo ethtool -s ens114f1np1 autoneg on
```

Ensure the link is up with
```bash
sudo ip link set dev ens114f1np1 down
sudo ip link set dev ens114f1np1 up
```

Wait for a while and verify that the link is detected on both sides with
```bash
ethtool <interface name>
```

<!-- For example, if you want to set up a 40G connection between interface A on the host and interface B on the DPU. On the host, run
```bash
sudo ethtool -s <interface A> autoneg off speed 40000 duplex full
```

On the DPU, follow section 5.3.2 of the user manual and run
```bash
i2cset -y 1 0x30 9 3
```
to enable the 100G port.  -->

# Running DPDK to test the PCIe connection

DPDK v19.11 (LTS) has been tested to work for this.

Follow instructions on the user manual section 8. Use the file `Helium-DPDK19.11-Vx.yRz.tar.gz` in the shared folder.

You might get some Werrors when running `make` to compile DPDK on the host. If you do, you can go into the relevant directory and remove the CFLAG `-Werror` from the Makefile. If you get a wrong pointer type warning, you can use typecasting to cast the pointer into the correct type.

There is one error where the patch can be referenced [here](https://git.dpdk.org/dpdk/commit/?id=87efaea6376c8). You can apply the patch manually.

On the host, run `lspci | grep b203`. Observe the first two devices have the addresses `0000:31:02.0` and `0000:31:02.1`. Bind these two virtual ports to DPDK.

Correction for command to run `./build/app/testpmd`: change the flag `-I` to `-i`.

# Running DPDK with Mellanox ports

For this, we use newer versions of DPDK to work with the MLX5 driver.

On the host, use DPDK v22.11 (LTS). On the DPU, use DPDK v20.11 (LTS) (newer versions have build issues).

The MLX5 port must not be bound to `vfio-pci` but left bound to `mlx5_core` (i.e. do not run `usertools/dpdk-devbind.py` to bind the device). There are a couple of sources for this, one of which is [here](https://www.jianshu.com/p/a415ad113615).

Install the latest NVIDIA MLNX_OFED from [here](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/). Also see the installation guide for DPDK MLX5 driver [here](https://doc.dpdk.org/guides/platform/mlx5.html#mlx5-common-compilation).

Install required libraries by running in the OFED directory

```bash
sudo ./mlnxofedinstall --dpdk --upstream-libs
```

On Poseidon, run testpmd with
```bash
sudo ./build/app/dpdk-testpmd -l 0,2,4 -a 0000:ca:00.1 -- -i  --rxd=4096
```

On the DPU, first bind the interface to use the `vfio-pci` driver. Then run testpmd with
```bash
./build/app/dpdk-testpmd -l 0,1-2 -w 0002:02:00.0 -- -i
```

If needed, enable hugepages on the DPU with
```bash
sysctl vm.nr_hugepages=12
```

Test packet flow between the two.

## Troubleshooting DPDK

You might encounter issues when running DPDK v22.11 on the host.

If you encounter `Unknown device` error in the binding, add the following lines to `dpdk-devbind.py` (around lines 40-50):

```python
cavium_sdp = {'Class': '0b', 'Vendor': '177d', 'Device': 'a303',
              'SVendor': None, 'SDevice': None}
octeontx2_sdp = {'Class': '08', 'Vendor': '177d', 'Device': 'a0f6,a0f7','SVendor': None, 'SDevice': None}
```

and modify line 67 to
```python
network_devices = [network_class, cavium_pkx, avp_vnic, ifpga_class, cavium_sdp, octeontx2_sdp]
```

Furthermore, modify `otx2_mbox.h` line 93 to
```c
#define OTX2_MBOX_VERSION (0x0009)
```

Follow user manual instructions to enable hugepages. Run testpmd with the following command:

```bash
./build/app/dpdk-testpmd -l0,1-2 -w 0002:02:00.0 -w 0002:0f:00.2 -w 0002:0f:00.3 -- -i --portmask=0x6
```
