# Known quirks of the wdk2023 Linux installation

## Why doesn't it boot from USB?

Well, it does now B-)

## The right firmware for WDK2023

The firmware appears to be partially specific to the wdk. It is contained in the windows installation, you can extract it from there.
![Screenshot of the location of sc8280xp firmware files](https://github.com/jglathe/linux_ms_dev_kit/assets/33185912/63ae339e-b963-4895-9204-406d8b3fbf5b)
@qzed mentioned that this can be due to model-specific signing. Firmwares that are actually loaded according to dmesg:
```
$ sudo dmesg|grep direct-loading
[    2.452023] firmware_class:fw_get_filesystem_firmware: platform regulatory.0: direct-loading regulatory.db
[    2.452247] firmware_class:fw_get_filesystem_firmware: platform regulatory.0: direct-loading regulatory.db.p7s
[    2.507062] firmware_class:fw_get_filesystem_firmware: remoteproc remoteproc0: direct-loading qcom/sc8280xp/MICROSOFT/DEVKIT23/qcadsp8280.mbn
[    2.551166] firmware_class:fw_get_filesystem_firmware: remoteproc remoteproc1: direct-loading qcom/sc8280xp/MICROSOFT/DEVKIT23/qccdsp8280.mbn
[    2.693501] firmware_class:fw_get_filesystem_firmware: mhi mhi0: direct-loading ath11k/WCN6855/hw2.1/amss.bin
[    2.810244] firmware_class:fw_get_filesystem_firmware: bluetooth hci0: direct-loading qca/hpbtfw21.tlv
[    2.906153] firmware_class:fw_get_filesystem_firmware: msm_dpu ae01000.display-controller: direct-loading qcom/a660_sqe.fw
[    2.906543] firmware_class:fw_get_filesystem_firmware: msm_dpu ae01000.display-controller: direct-loading qcom/a690_gmu.bin
[    2.910708] firmware_class:fw_get_filesystem_firmware: msm_dpu ae01000.display-controller: direct-loading qcom/sc8280xp/MICROSOFT/DEVKIT23/qcdxkmsuc8280.mbn
[    3.424541] firmware_class:fw_get_filesystem_firmware: bluetooth hci0: direct-loading qca/hpnv21g.bin
[    3.437538] firmware_class:fw_get_filesystem_firmware: ath11k_pci 0006:01:00.0: direct-loading ath11k/WCN6855/hw2.1/board-2.bin
[    3.453680] firmware_class:fw_get_filesystem_firmware: ath11k_pci 0006:01:00.0: direct-loading ath11k/WCN6855/hw2.1/regdb.bin
[    3.460294] firmware_class:fw_get_filesystem_firmware: ath11k_pci 0006:01:00.0: direct-loading ath11k/WCN6855/hw2.1/board-2.bin
[    3.487535] firmware_class:fw_get_filesystem_firmware: ath11k_pci 0006:01:00.0: direct-loading ath11k/WCN6855/hw2.1/m3.bin
[    6.071910] firmware_class:fw_get_filesystem_firmware: r8152 6-1.1:1.0: direct-loading rtl_nic/rtl8153b-2.fw
[    9.043196] firmware_class:fw_get_filesystem_firmware: bluetooth hci0: direct-loading qca/hpbtfw21.tlv
[    9.672265] firmware_class:fw_get_filesystem_firmware: bluetooth hci0: direct-loading qca/hpnv21g.bin
```
As it turns out qcdxkmsuc8280.mbn and qcvss8280.mbn (which is not loaded) belong to the Windows Graphics driver. The dts advises to load qcdxkmsuc8280.mbn, and this appears to work. qcvss8280.mbn appears to belong to HDCP according to the windows driver .inf, no idea if it can be useful.
The target directory for the remoteproc fw is important, it doesn't exist. It needs to be created first.
```
sudo mkdir -p /lib/firmware/qcom/sc8280xp/MICROSOFT/DEVKIT23/
```

## Power Management is in disarray

To keep the thing working almost all voltage regulators are set to always on. If you browse in the sc8280xp code, you'll see that power management isn't supported yet by qcom-pcie, leading to the ```pcie_3a_gdsc status stuck at 'off'``` kernel warning. With no further effects, only that pcie3a where WWAN lives is disbled. Likewise, qcom-pmic-glink doesn't get the links it needs to work:
```
[    2.049181] qcom_pmic_glink pmic-glink: Failed to create device link (0x180) with usb0-sbu-mux
[    2.056263] qcom_pmic_glink pmic-glink: Failed to create device link (0x180) with usb1-sbu-mux
[    2.230842] qcom_pmic_glink pmic-glink: Failed to create device link (0x180) with a600000.usb
[    2.248853] qcom_pmic_glink pmic-glink: Failed to create device link (0x180) with ae90000.displayport-controller
[    2.251678] qcom_pmic_glink pmic-glink: Failed to create device link (0x180) with ae98000.displayport-controller
[    2.944111] qcom_pmic_glink pmic-glink: Failed to create device link (0x180) with a800000.usb
```
Also, if you happen to use an rsyslogd server to monitor devices, this will show up in it, every 30 seconds:
```
Jun 23 18:55:17 snapdragix upowerd[1141]: Setting /sys/devices/platform/pmic-glink/pmic_glink.power-supply.0/power_supply/qcom-battmgr-bat state empty as unknown and very low
Jun 23 18:55:17 snapdragix upowerd[1141]: Setting /sys/devices/platform/pmic-glink/pmic_glink.power-supply.0/power_supply/qcom-battmgr-usb state empty as unknown and very low
Jun 23 18:55:17 snapdragix upowerd[1141]: Setting /sys/devices/platform/pmic-glink/pmic_glink.power-supply.0/power_supply/qcom-battmgr-wls state empty as unknown and very low
Jun 23 18:55:47 snapdragix upowerd[1141]: Setting /sys/devices/platform/pmic-glink/pmic_glink.power-supply.0/power_supply/qcom-battmgr-bat state empty as unknown and very low
Jun 23 18:55:47 snapdragix upowerd[1141]: Setting /sys/devices/platform/pmic-glink/pmic_glink.power-supply.0/power_supply/qcom-battmgr-usb state empty as unknown and very low
Jun 23 18:55:47 snapdragix upowerd[1141]: Setting /sys/devices/platform/pmic-glink/pmic_glink.power-supply.0/power_supply/qcom-battmgr-wls state empty as unknown and very low
```
Since the wdk doesn't have a battery this is understandable. You can filter it out, though. On the wdk, in /etc/rsyslog.conf, add following lines add the end:
```
#
# Include all config files in /etc/rsyslog.d/
#
$IncludeConfig /etc/rsyslog.d/*.conf
#------- this is the normal end, add from here

# special rules
# filter out upowerd message that there is no battery and things are strange
if ($msg contains "state empty as unknown and very low") then {
        stop
}

# send the logs away
*.* @yoursyslogserver:514

```
The important part is that the target is specified after all rules you want to apply.
The wdk also thinks it is on battery power (not AC), and disables the ```apt``` update tasks. 

## Sound over DP doesn't work yet, USB sound devices do, though
