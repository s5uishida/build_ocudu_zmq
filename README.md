# Build OCUDU 5G RAN with ZeroMQ
[OCUDU](https://gitlab.com/ocudu/ocudu) is a permissively-licensed, open-source 5G (and beyond) CU/DU project designed for commercial deployment and broad industry adoption, as well as advanced research and development. Therefore, in order to confirm the facilities of 5GC, I will describe the simple procedure for building the virtual gNodeB instead of the real device.

Please refer to the following for building OCUDU 5G RAN with ZeroMQ.
- Installation Guide - https://ocudu-docs-604e90.gitlab.io/user_manual/installation/
- ZeroMQ-based Setup - https://ocudu-docs-604e90.gitlab.io/user_manual/tutorials/srsue/#zeromq-based-setup
- Configuration Reference - https://ocudu-docs-604e90.gitlab.io/user_manual/config_reference/#configuration-reference-1

The specification of the VM that have been confirmed to work is as follows.
| OS | CPU (Min) | Memory (Min) | HDD (Min) |
| --- | --- | --- | --- |
| Ubuntu 24.04 | 5 | 4GB | 10GB |

**4GB or more memory is required to build. And when OCUDU gNodeB is configured with the ZMQ-based RF driver, 5 CPU cores or more are required to start up and accept connections from UEs. This constraint regarding the minimum number of CPU cores may be unique to my environment.**

Also, when connecting by 5G NR-UE with ZeroMQ, see [here](https://github.com/s5uishida/build_srsran_4g_zmq_disable_rf_plugins) for how to build and configure this RF simulated UE.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Install the required libraries including ZeroMQ](#install_libs)
- [Clone OCUDU](#clone_ocudu)
- [Build OCUDU 5G RAN](#build)
- [Create the configuration file of gNodeB](#create_gnb_config)
  - [Add a Slice configuration](#add_slice)
- [Issues](#issues)
- [Confirmed Version List](#ver_list)
- [Sample Configurations](#sample_conf)
- [Changelog (summary)](#changelog)

---

<a id="install_libs"></a>

## Install the required libraries including ZeroMQ

```
# apt install cmake make gcc g++ pkg-config libfftw3-dev libmbedtls-dev libsctp-dev libyaml-cpp-dev libgtest-dev libzmq3-dev
```

<a id="clone_ocudu"></a>

## Clone OCUDU

```
# git clone https://gitlab.com/ocudu/ocudu.git
```

<a id="build"></a>

## Build OCUDU 5G RAN

```
# cd ocudu
# mkdir build
# cd build
# cmake ../ -DENABLE_EXPORT=ON -DENABLE_ZEROMQ=ON
# make -j`nproc`
```
If you do not want to build `tests` target, add `-BUILD_TESTING=OFF` option.
```
# cmake ../ -DENABLE_EXPORT=ON -DENABLE_ZEROMQ=ON -DBUILD_TESTING=OFF
```

<a id="create_gnb_config"></a>

## Create the configuration file of gNodeB

Get [gNB config](https://docs.srsran.com/projects/project/en/latest/_downloads/a7c34dbfee2b765503a81edd2f02ec22/gnb_zmq.yaml) of [ZeroMQ-based Setup](https://docs.srsran.com/projects/project/en/latest/tutorials/source/srsUE/source/index.html#zeromq-based-setup) for OCUDU as the original file.
```
# cd ocudu/build/apps/gnb
# wget <link of "gNB config">
```
For reference, `gnb_zmq.yaml` on 2025.01.15 is as follows.
```yaml
# This configuration file example shows how to configure the srsRAN Project gNB to allow srsUE to connect to it. 
# This specific example uses ZMQ in place of a USRP for the RF-frontend, and creates an FDD cell with 10 MHz bandwidth. 
# To run the srsRAN Project gNB with this config, use the following command: 
#   sudo ./gnb -c gnb_zmq.yaml

cu_cp:
  amf:
    addr: 10.53.1.2                 # The address or hostname of the AMF.
    port: 38412
    bind_addr: 10.53.1.1            # A local IP that the gNB binds to for traffic from the AMF.
    supported_tracking_areas:
      - tac: 7
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
              - sst: 1
  inactivity_timer: 7200            # Sets the UE/PDU Session/DRB inactivity timer to 7200 seconds. Supported: [1 - 7200].

ru_sdr:
  device_driver: zmq                # The RF driver name.
  device_args: tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,base_srate=23.04e6 # Optionally pass arguments to the selected RF driver.
  srate: 23.04                      # RF sample rate might need to be adjusted according to selected bandwidth.
  tx_gain: 75                       # Transmit gain of the RF might need to adjusted to the given situation.
  rx_gain: 75                       # Receive gain of the RF might need to adjusted to the given situation.

cell_cfg:
  dl_arfcn: 368500                  # ARFCN of the downlink carrier (center frequency).
  band: 3                           # The NR band.
  channel_bandwidth_MHz: 20         # Bandwith in MHz. Number of PRBs will be automatically derived.
  common_scs: 15                    # Subcarrier spacing in kHz used for data.
  plmn: "00101"                     # PLMN broadcasted by the gNB.
  tac: 7                            # Tracking area code (needs to match the core configuration).
  pdcch:
    common:
      ss0_index: 0                  # Set search space zero index to match srsUE capabilities
      coreset0_index: 12            # Set search CORESET Zero index to match srsUE capabilities
    dedicated:
      ss2_type: common              # Search Space type, has to be set to common
      dci_format_0_1_and_1_1: false # Set correct DCI format (fallback)
  prach:
    prach_config_index: 1           # Sets PRACH config to match what is expected by srsUE
  pdsch:
    mcs_table: qam64                # Sets PDSCH MCS to 64 QAM
  pusch:
    mcs_table: qam64                # Sets PUSCH MCS to 64 QAM

log:
  filename: /tmp/gnb.log            # Path of the log file.
  all_level: info                   # Logging level applied to all layers.
  hex_max_size: 0

pcap:
  mac_enable: false                 # Set to true to enable MAC-layer PCAPs.
  mac_filename: /tmp/gnb_mac.pcap   # Path where the MAC PCAP is stored.
  ngap_enable: false                # Set to true to enable NGAP PCAPs.
  ngap_filename: /tmp/gnb_ngap.pcap # Path where the NGAP PCAP is stored.
```
Then, edit `gnb_zmq.yaml` with reference to [this](https://ocudu-docs-604e90.gitlab.io/user_manual/config_reference/#configuration-reference-1) according to your environment.
When setting the IP address of the N3 interface, add the following parameter and set the appropriate IP address.
```diff
--- gnb_zmq.yaml.orig   2025-01-15 18:27:10.000000000 +0900
+++ gnb_zmq.yaml        2026-02-20 10:19:35.831332416 +0900
@@ -16,6 +16,13 @@
               - sst: 1
   inactivity_timer: 7200            # Sets the UE/PDU Session/DRB inactivity timer to 7200 seconds. Supported: [1 - 7200].
 
+cu_up:
+  ngu:
+    socket:                               # Define socket(s) for NG-U interface.
+      - bind_addr: 127.0.3.1              # Optional TEXT (auto). Sets local IP address to bind for N3 interface. Format: IPV4 or IPV6 IP address.
+        bind_interface: auto              # Optional TEXT (auto). Network device to bind for N3 interface
+        ext_addr: auto                    # Optional TEXT (auto). Sets external IP address that is advertised to receive GTP-U packets from UPF via N3 interface.
+
 ru_sdr:
   device_driver: zmq                # The RF driver name.
   device_args: tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,base_srate=23.04e6 # Optionally pass arguments to the selected RF driver.
```

<a id="add_slice"></a>

### Add a Slice configuration

The following SST/SD values are in decimal notation.
For example, SST=0x1 and SD=0x010203 are expressed in decimal as follows.
```diff
cu_cp:
  amf:
    addr: 10.53.1.2                 # The address or hostname of the AMF.
    port: 38412
    bind_addr: 10.53.1.1            # A local IP that the gNB binds to for traffic from the AMF.
    supported_tracking_areas:
      - tac: 7
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
-->           - sst: 1
-->             sd: 66051
  inactivity_timer: 7200            # Sets the UE/PDU Session/DRB inactivity timer to 7200 seconds. Supported: [1 - 7200].
```

<a id="issues"></a>

## Issues

If the latest `main` branch doesn't work, you may try the latest `release` version.
**When OCUDU gNodeB is configured with the ZMQ-based RF driver, this constraint regarding the minimum number of CPU cores may be unique to my environment.**

1. 5 CPU cores or more are required to start up and accept connections from UEs.

<a id="ver_list"></a>

## Confirmed Version List

I simply confirmed the operation of the following versions.

| Version | Commit | Date | Issues |
| --- | --- | --- | -- |
| -- | `b8e3be1e489dcbec3bb3c12e4d291c1d95b9cec0` | 2026.02.19 | 1 |

<a id="sample_conf"></a>

## Sample Configurations

- [Open5GS 5GC & srsRAN 5G with ZeroMQ UE / RAN Sample Configuration](https://github.com/s5uishida/open5gs_5gc_srsran_sample_config)
- [free5GC 5GC & srsRAN 5G with ZeroMQ UE / RAN Sample Configuration](https://github.com/s5uishida/free5gc_srsran_sample_config)

<a id="changelog"></a>

## Changelog (summary)

- [2026.02.20] Initial release.
