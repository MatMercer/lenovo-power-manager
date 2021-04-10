# Lenovo Power Manager
A power manager for Lenovo notebooks using simple ACPI calls.

# Disclaimer

⚠️ I am quite new to the matter and experimenting about the performance modes available. I'm not filliated with Lenovo in any way and don't take responsibility for possible damages of the devices by thermal means.

⚠️ Beware that most Linux Kernels don't implement "Lap mode" [as described in this document](https://web.archive.org/web/20190926163900/https://forums.lenovo.com/lnv/attachments/lnv/Special_Interest_Linux/13642/1/Linux%20Thermal%20throttling.pdf). This means you can get burns 🔥 in your lap if you use the performance mode while using the notebook in your lap! Linux has no way of detecting the device is in your lap most of the times. 

# Currently supported devices

|Model|Power States|
|---|---|
|Thinkpad T14 Gen 1|economy balanced performance|

# TODO's

🏃 [Check the management board of the project](https://github.com/MatMercer/lenovo-power-manager/projects)

# 🧰 Motivation and Technical Details

## 🐌 The Linux struggle with Intel DPTF

DPTF is a technology for managing thermal performance of mobile devices by Intel. It is closed source and quite recent. So there isn't many information about it out there.

Current implementations of DPTF are very small for Linux, meanwhile it is working fine for Windows. The software for managing the DPTF for Lenovo in Windows is called [Lenovo Vantage](https://www.lenovo.com/us/en/software/vantage) and it does a very good job. Lenovo Vantage adds 3 power consumption profiles in the "battery settings" in your taskbar:

|Lenovo Performance States|
|---|
|economy|
|balanced|
|performance|

TODO: print of it

Sadly, there isn't such thing yet to configure these performance states, so **the main motivation of this project is to make the management of power profiles for Lenovo devices easier in Linux**. The final goal is to have a applet in multiple DE's to manage power comsumption, just like in Windows.

## Why not using perfomance management is bad

There are 2 reasons performance profiles must be managed:

1. Battery life
2. Performance Throttling

Talking about the reason 2, the balanced mode doesn't alow you to use the full power of your device. Nvidia drivers report throlttling at 57 celsius sometimes! And Intel CPU's sometimes throttles to very low performance levels.

## How Windows manages the power states

I really don't know exactly. To know that, I would need to reverse engineer Lenovo Vantage, which is a no-no. But, [an entry in the ArchWiki gave me some light](https://wiki.archlinux.org/index.php/Lenovo_IdeaPad_5_15are05) about the matter.

**Power management of Intel DPTF is managed by ACPI calls**, described in the wiki.

## Finding the right ACPI call for your device

ACPI calls are quite a mess, weird and sometimes very hard to understand, since almost everything is closed source. The main struggle with that is the fact that **every Lenovo model may have a different ACPI call for the power profiles**.

I have succesfully found the ACPI call of my device by:

Ectracting the ACPI binary:

```
mkdir ~/acpidump
cd ~/acpidump

acpidump > acpidata.dat
acpiextract -sSSDT acpidata.dat
acpiextract -sDSDT acpidata.dat
```

Finding "hints" about performance modes. (hex values found in ArchWiki)

```
grep -Ri "000FB001" .
./dsdt.dsl:            ^HKEY.DYTC (0x000FB001)

grep -Ri "0012B001" .
./dsdt.dsl:                ^HKEY.DYTC (0x0012B001)

grep -Ri "0013B001" .
./dsdt.dsl:                ^HKEY.DYTC (0x0013B001)
```

With that, I was able to find the 3 performance ACPI methods and unlock the max performance mode:

```
c(){ echo "$1" | sudo tee /proc/acpi/call >/dev/null && sudo cat /proc/acpi/call;echo;}

c '\_SB.PCI0.LPCB.EC._Q6D'
```

> ⚠️ The ACPI call above is just an example and may not work in your device

## Checking if the performance mode was activated

### Use your ears 👂

The fan of your device will be louder if you activate the performance mode and quieter if you activate the economy mode.

### Check for max temperature allowed for the GPU (NVidia Only)

If the command:

```
nvidia-settings -q GPUMaxOperatingTempThreshold
  Attribute 'GPUMaxOperatingTempThreshold' (redacted:0[gpu:0]): 57.
    'GPUMaxOperatingTempThreshold' is an integer attribute.
    'GPUMaxOperatingTempThreshold' is a read-only attribute.
    'GPUMaxOperatingTempThreshold' can use the following target types: X Screen, GPU.
```

Reports 57 or a number close to that, it means the device isn't in performance mode.

Example of performance mode output:

```
nvidia-settings -q GPUMaxOperatingTempThreshold
  Attribute 'GPUMaxOperatingTempThreshold' (redacted:0[gpu:0]): 87.
    'GPUMaxOperatingTempThreshold' is an integer attribute.
    'GPUMaxOperatingTempThreshold' is a read-only attribute.
    'GPUMaxOperatingTempThreshold' can use the following target types: X Screen, GPU.
```

The output above is stating that it will alow 87C for the GPU, which is a good sign.
