# HP Chromebook 11 G6 EE (SNAPPY) Audio Fix Runbook

**Hardware:** HP Chromebook 11 G6 EE (Intel Apollo Lake Celeron N3350, board: `SNAPPY`)  
**Firmware:** MrChromebox Full ROM UEFI  
**Operating System:** Linux Mint XFCE (PipeWire / WirePlumber audio stack)

This guide documents the complete, reproducible process to fix missing audio ("Dummy Output") and configure the internal stereo speakers (MAX98357A), 3.5mm headset jack (DA7219), and digital microphones (DMIC) using the modern Intel AVS kernel driver stack.

---

## 1. Prerequisites & Required Tools

Ensure necessary decompression utilities, package downloaders, and audio management packages are installed:

```bash
sudo apt update
sudo apt install -y curl zstd alsa-utils pavucontrol
```

---

## 2. Force the Intel AVS Driver in GRUB

The default Sound Open Firmware (`dsp_driver=3`) and legacy HD-Audio (`dsp_driver=1`) drivers fail to initialize the audio cluster on Apollo Lake Chromebooks. Switch to the modern Intel AVS driver stack (`dsp_driver=4`):

1. Open `/etc/default/grub` in a text editor:
   ```bash
   sudo nano /etc/default/grub
   ```

2. Locate `GRUB_CMDLINE_LINUX_DEFAULT` and append `snd_intel_dspcfg.dsp_driver=4`.  
   Example:
   ```ini
   GRUB_CMDLINE_LINUX_DEFAULT="quiet splash sdhci.debug_quirks2=0x4 mem_sleep_default=deep snd_intel_dspcfg.dsp_driver=4"
   ```

3. Rebuild the GRUB bootloader configuration:
   ```bash
   sudo update-grub
   ```

---

## 3. Deploy DSP Base Firmware & Audio Topologies

The `snd_soc_avs` kernel driver requires base DSP firmware placed under `/lib/firmware/intel/avs/apl/` and board topologies under `/lib/firmware/intel/avs/`.

### A. Decompress and Link Apollo Lake Base DSP Firmware

Extract the pre-existing system Broxton/Apollo Lake DSP binary into the path expected by the AVS driver:

```bash
sudo mkdir -p /lib/firmware/intel/avs/apl

# Extract and link base DSP firmware
sudo zstd -d -c /lib/firmware/intel/dsp_fw_bxtn_v3366.bin.zst | sudo tee /lib/firmware/intel/avs/apl/dsp_basefw.bin > /dev/null
sudo ln -sf /lib/firmware/intel/avs/apl/dsp_basefw.bin /lib/firmware/intel/avs/apl/dsp.bin
sudo cp /lib/firmware/intel/dsp_fw_bxtn_v3366.bin.zst /lib/firmware/intel/avs/apl/dsp_basefw.bin.zst
sudo ln -sf /lib/firmware/intel/avs/apl/dsp_basefw.bin.zst /lib/firmware/intel/avs/apl/dsp.bin.zst
```

### B. Download Upstream Audio Topologies

Download the upstream topology binaries matching the hardware codecs on the `SNAPPY` motherboard:

```bash
cd /lib/firmware/intel/avs

sudo curl -fSL -o max98357a-tplg.bin "[https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/intel/avs/max98357a-tplg.bin](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/intel/avs/max98357a-tplg.bin)"
sudo curl -fSL -o da7219-tplg.bin "[https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/intel/avs/da7219-tplg.bin](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/intel/avs/da7219-tplg.bin)"
sudo curl -fSL -o dmic-tplg.bin "[https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/intel/avs/dmic-tplg.bin](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/intel/avs/dmic-tplg.bin)"

# Symlink legacy aliases
sudo ln -sf max98357a-tplg.bin avs_max98357a.bin 2>/dev/null
sudo ln -sf da7219-tplg.bin avs_da7219.bin 2>/dev/null

# Remove broken HDMI topologies that cause probe failures (-22)
sudo rm -f /lib/firmware/intel/avs/hda*
```

---

## 4. Reboot & Verify Hardware Detection

Restart the machine to boot with the new kernel parameters and firmware:

```bash
sudo reboot
```

After logging back into your desktop, verify that the kernel and ALSA have instantiated the hardware:

1. **Verify driver binding:**
   ```bash
   lspci -nnk -s 00:0e.0
   ```
   *Expected result:* `Kernel driver in use: snd_soc_avs`

2. **Verify registered ALSA sound cards:**
   ```bash
   cat /proc/asound/cards
   ```
   *Expected result:* Lists three functional cards:
   * `card 1: avsdmic`
   * `card 2: avsda7219`
   * `card 3: avsmax98357a`

3. **Verify PCM playback streams:**
   ```bash
   aplay -l
   ```
   *Expected result:* Confirms device endpoints at index `1`:
   * `card 2: avsda7219 [...], device 1: Headset`
   * `card 3: avsmax98357a [...], device 1: Built-in Speakers`

---

## 5. Direct ALSA Playback Test

Test direct hardware playback through the Maxim internal speaker amplifier on subdevice 1 (`plughw:3,1`):

```bash
speaker-test -D plughw:3,1 -t wav -c 2
```

* **Verification:** You should clearly hear "Front Left", "Front Right" spoken through the laptop speakers. Press `Ctrl + C` to stop.

---

## 6. Desktop Session & WirePlumber Activation

Because the physical amplifier uses subdevice index `1`, WirePlumber defaults to "Off" or "Dummy Output" until the active profile is selected.

### A. Clear Cached Audio Session State

Flush any stale or dead session daemons:

```bash
systemctl --user stop pipewire.socket pipewire-pulse.socket wireplumber pipewire pipewire-pulse
rm -rf ~/.local/state/wireplumber ~/.local/state/pipewire
systemctl --user start pipewire.socket pipewire-pulse.socket wireplumber pipewire pipewire-pulse
```

### B. Activate Card Profile in Pavucontrol

1. Open PulseAudio Volume Control:
   ```bash
   pavucontrol &
   ```
2. Navigate to the **Configuration** tab.
3. Locate `avs_max98357a` (or Built-in Audio) and switch the dropdown menu from **Off** to **Play HiFi quality Music** or **Stereo Output / Pro Audio**.
4. Go to the **Output Devices** tab, locate **Built-in Speakers**, and click the green checkmark icon (**"Set as fallback"**).

### C. Set Default Sink via Terminal (Alternative)

To set the default sink and volume from the command line:

1. Identify the speaker sink ID:
   ```bash
   wpctl status
   ```
2. Set the default sink and adjust baseline levels:
   ```bash
   wpctl set-default <SINK_ID>
   wpctl set-volume @DEFAULT_AUDIO_SINK@ 70%
   wpctl set-mute @DEFAULT_AUDIO_SINK@ 0
   ```
   *(Replace `<SINK_ID>` with the numeric ID found under `Audio -> Sinks` in `wpctl status`).*

### D. Persist ALSA State

Save the unmuted mixer state to survive future boots:

```bash
sudo alsactl store
sync
```

---

## 7. Speaker Protection & Volume Limiting

> **Hardware Warning:** ChromeOS uses proprietary software DSP compressors and high-pass filters to protect Chromebook speakers. Under Linux, the Maxim MAX98357A amplifier passes digital signals directly to the transducers with fixed gain. Running internal speakers at 100% (0 dBFS) can introduce severe distortion, overheat the voice coils, or permanently tear the speaker cones.

To protect the hardware, lock a hard digital volume ceiling of **70%–75%** inside WirePlumber.

### A. Create the WirePlumber Configuration Rule

Create a persistent configuration file:

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d/

cat << 'EOF' > ~/.config/wireplumber/wireplumber.conf.d/51-limit-speakers.conf
monitor.alsa.rules = [
  {
    matches = [
      {
        node.name = "~alsa_output.*max98357a.*"
      }
    ]
    actions = {
      update-props = {
        node.volume = 0.70
        node.max-volume = 0.75
      }
    }
  }
]
EOF
```

### B. Apply the Configuration

Restart the session manager:

```bash
systemctl --user restart wireplumber
```

* **Verification:** The desktop volume slider and multimedia keys will now treat 75% as the maximum ceiling, keeping the speakers safe from blowouts.
* 
