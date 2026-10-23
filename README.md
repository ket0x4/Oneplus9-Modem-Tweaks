# Qualcomm Snapdragon X60 Modem Tweaks & 5G SA Unlock

A comprehensive, reproducible guide to unlocking the full hardware and modem capabilities of Qualcomm Snapdragon X60 baseband modems (tested on **OnePlus 9 / LE2115** running AOSP/crDroid).

---

> [!CAUTION]
> ### Legal Disclaimer & Liability Notice
> This guide and its contents are provided strictly for **educational, academic, and technical research purposes**.
> 
> 1. **No Liability ("As-Is"):** Modifying modem non-volatile (NV) memory, baseband partitions, and low-level EFS items carries inherent risks. The author assumes **zero liability** for bricked devices, corrupted EFS/modem partitions, permanent loss of baseband/IMEI, voided hardware warranties, or any physical or operational damages resulting from these modifications.
> 2. **Telecommunications Regulatory Compliance:** Radio frequency emissions, certified band capabilities, and transmit power limits are governed by telecommunications authorities (e.g., FCC in the US, CE/RED in the EU, BTK in Turkey). Overriding OEM carrier policies or transmitting on unauthorized frequency bands may violate local or national laws in your jurisdiction.
> 3. **Warranty Void:** Rooting your device, unlocking bootloaders, and injecting non-certified radio parameters permanently voids any manufacturer or carrier warranty.
> 4. **User Responsibility:** You are solely responsible for ensuring compliance with your country’s radio frequency and telecommunications regulations before applying any tweaks.

---

## TL;DR

OnePlus ships its NR5G mode as **"NSA only"** on various regional variants, restricting Feature Group Indicators (FGI), VoNR, and throughput via modem EFS throttling. By flipping these NV items and updating `carrier_policy.xml`, we unlock full hardware bands, 5G SA/NSA, advanced modulations (256QAM DL / 64QAM UL), and remove data throttle caps.

> [!NOTE]
> This guide was created and verified on the **OnePlus 9 (LE2115 - North America model)**. The procedures apply to other regional variants (including OnePlus 9 Pro) and general Qualcomm Snapdragon 888 (X60) platforms with minor path variations.

---

## Requirements

* **Root Access:** Magisk or KernelSU installed.
* **Tool:** [QCSuper](https://github.com/p1sec/qcsuper) installed on host machine.
* **Environment:** Android Platform Tools (`adb`) installed with USB Debugging enabled.

> [!IMPORTANT]
> Modem EFS write operations require an exclusive lock on the Qualcomm DIAG port. If you have **Network Signal Guru (NSG)** installed, force-stop or disable it before proceeding:
> ```bash
> adb shell su -c "pm disable-user --user 0 com.qtrun.QuickTest"
> ```

---

## Step-by-Step Guide

### Step 1: Backup Critical Modem & EFS Partitions

Run these commands in your **host terminal** to dump block-level images of your baseband and calibration partitions before making any modifications:

```bash
# Create local backup directory
mkdir -p backup

# Dump modem partitions
adb shell su -c "dd if=/dev/block/by-name/modemst1 of=/sdcard/modemst1.img bs=4096"
adb shell su -c "dd if=/dev/block/by-name/modemst2 of=/sdcard/modemst2.img bs=4096"
adb shell su -c "dd if=/dev/block/by-name/fsg of=/sdcard/fsg.img bs=4096"
adb shell su -c "dd if=/dev/block/by-name/fsc of=/sdcard/fsc.img bs=4096"
adb shell su -c "dd if=/dev/block/by-name/modem_a of=/sdcard/modem_a.img bs=4096"
adb shell su -c "dd if=/dev/block/by-name/modem_b of=/sdcard/modem_b.img bs=4096"

# Pull dumps to computer
adb pull /sdcard/modemst1.img backup/
adb pull /sdcard/modemst2.img backup/
adb pull /sdcard/fsg.img backup/
adb pull /sdcard/fsc.img backup/
adb pull /sdcard/modem_a.img backup/
adb pull /sdcard/modem_b.img backup/

# Clean up SD card
adb shell "rm /sdcard/*.img"
```

---

### Step 2: Prepare Payloads & Expose DIAG Port

Run these commands in your **host terminal**:

```bash
# 1. Create binary byte payloads on host
printf '\x00' > zero.bin
printf '\x01' > one.bin

# 2. Reboot to Factory Test Mode (required for oneplus 9)
adb reboot ftm

# 3. Set SELinux to permissive and expose Qualcomm DIAG endpoint
adb shell su -c "setenforce 0"
adb shell su -c "setprop sys.usb.config diag,adb"

# 4. Launch QCSuper EFS interactive shell
qcsuper --adb --efs-shell
```

---

### Step 3: Apply Modem EFS Tweaks

All commands in this section must be executed inside the **QCSuper EFS Shell** (`>>>` prompt):

#### 1. Unlock 5G SA & Handover Stability (MMODE NV Items)
Forces NSA + SA dual-mode capability and prevents the modem from aggressively downgrading from SA to LTE:

```bash
put zero.bin /nv/item_files/modem/mmode/nr5g_disable_mode
put zero.bin /nv/item_files/modem/mmode/nr5g_disable_mode_Subscription01
put zero.bin /nv/item_files/modem/mmode/downgrade_sa
put zero.bin /nv/item_files/modem/mmode/downgrade_sa_Subscription01
```

> [!NOTE]
>  If you want your device to automatically handle transitions between 5G Standalone (SA) and Non-Standalone (NSA) modes rather than forcing a single state:
> ```
> put one.bin /nv/item_files/modem/mmode/downgrade_sa
> put one.bin /nv/item_files/modem/mmode/downgrade_sa_Subscription01
> ```

#### 2. Unlock FGI & Modulation Constraints (RRC Cap)
Resets feature group indicator restrictions to allow 256QAM DL, 64QAM UL, and broader Carrier Aggregation:

```bash
put zero.bin /nv/item_files/modem/lte/rrc/cap/diff_fdd_tdd_fgi_enable
put zero.bin /nv/item_files/modem/lte/rrc/cap/diff_fdd_tdd_fgi_enable_Subscription01
```

#### 3. Bypass APN & Data Throttle Limits
Removes internal baseband PDN penalty timers and throughput clamp profiles:

```bash
put zero.bin /apn_throttle/pdn_limit_throttle_params.txt
put zero.bin /apn_throttle/apn_throttle_params
```

#### 4. Carrier Policy Band & Full CA Unlocking
Unlocks all hardware RF bands and Carrier Aggregation matrices by updating `carrier_policy.xml`:

```bash
# Backup existing policy
get /policyman/carrier_policy.xml carrier_policy_backup.xml

# Flash new policy (ensure carrier_policy.xml is in your working directory)
put carrier_policy.xml /policyman/carrier_policy.xml
```

#### 5. Force Voice over 5G NR (VoNR)
Prevents EPS fallback to LTE during voice calls on 5G Standalone networks:

```bash
put one.bin /nv/item_files/modem/mmode/nr5g_full_voice_support
put one.bin /nv/item_files/modem/mmode/nr5g_full_voice_support_Subscription01
```

#### 6. (Optional) Disable Dynamic SAR Power Throttling
Zeros out the 145-byte OEM Device State Index (DSI) back-off table to maintain maximum uplink RF budget (+23 dBm to +26 dBm PUSCH Tx power) in fringe areas:

> [!WARNING]
> **Health & Hardware Safety Warning:**
> Specific Absorption Rate (SAR) limits are legally mandated thresholds (FCC, CE, ICNIRP) that protect human tissue from excessive RF energy. Disabling this table forces the modem to transmit at **maximum RF power at all times**, completely ignoring proximity sensors, head detection during calls, and touch state.
> * **Health:** Exceeds government-certified RF exposure limits when held in hand, placed against the head, or kept in pockets.
> * **Hardware:** Sustained peak transmission causes accelerated battery wear, increased device temperatures, and elevated thermal stress on the RF front-end (PA/FEM chips).
> 
> *Leave untouched if regulatory compliance or thermal safety is prioritized.*

```bash
# Backup stock SAR config
get /nv/item_files/rfnv/oem_sar_dsi_config oem_sar_dsi_config.bak

# Disable SAR back-off
put zero.bin /nv/item_files/rfnv/oem_sar_dsi_config
```

#### 7. Exit EFS Shell
Once all files are written, exit the QCSuper prompt:

```bash
exit
```
*(Or press `Ctrl + D`)*

---

### Step 4: Apply Android Telephony Properties & Reboot

Run these final commands from your **host terminal** to configure the Android OS telephony framework:

```bash
# Enable VoNR in Android telephony stack
adb shell su -c "setprop persist.radio.vonr.enabled true"
adb shell su -c "setprop persist.vendor.radio.vonr_enabled 1"

# Restore SELinux to Enforcing
adb shell su -c "setenforce 1"

# Reboot device to apply baseband configuration
adb reboot
```

---

## Verification

After the device boots, wait approximately 60 seconds and run:

```bash
# Check active radio technology
adb shell dumpsys telephony.registry | grep -oE 'getRilDataRadioTechnology=[0-9]+\([A-Z_]+\)' | tail -1

# Expected outputs:
# 20(NR_SA)  -> Standalone 5G Active
# 14(LTE)    -> LTE Active

# Check network overrides
adb shell dumpsys telephony.registry | grep -oE 'network=[A-Z_]+, overrideNetwork=[A-Z_]+' | tail -1

# Expected outputs:
# network=NR, overrideNetwork=NONE     -> True 5G SA
# network=LTE, overrideNetwork=NR_NSA  -> 5G NSA
```
