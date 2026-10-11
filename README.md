# Unlocking Full Fan Performance (3900+ RPM) on Lenovo ThinkPad E15 Gen 4 (and similar) under Linux

A comprehensive guide and technical post-mortem on reverse engineering the ACPI firmware of the **Lenovo ThinkPad E15 Gen 4 (Intel)** to bypass the hard-coded $\sim 3100\text{--}3300\text{ RPM}$ fan speed limit under Linux.

Tested and verified on **Ubuntu 24.04.4 LTS** (Linux Kernel 7.0+).

## 1. Problem Overview

Under Microsoft Windows with Lenovo Vantage and Intel drivers installed, the ThinkPad E15 Gen 4 fan easily spins up to $3900\text{+ RPM}$ under heavy sustained CPU workloads.

Under Linux, however:

* The fan remains artificially throttled to $\sim 3100\text{--}3300\text{ RPM}$, regardless of thermal pressure ($90^\circ\text{C}+$ on CPU package).

* Traditional utilities like `thinkfan` or writing to `/proc/acpi/ibm/fan` fail to push the fan beyond this ceiling because the Embedded Controller (EC) firmware overrides OS commands.

* Heavy computational tasks (compilation, container builds, gaming) suffer from severe thermal throttling.

* **The Clue:** If the laptop was booted into Windows and then warm-rebooted into Linux, the fan could reach $3900\text{+ RPM}$. However, after a cold boot, shutdown, or S3/S0ix sleep cycle, the fan fell back to the $\sim 3100\text{ RPM}$ ceiling.

## 2. Root Cause Analysis & ACPI Reverse Engineering

Disassembling the DSDT and SSDT tables (`iasl -d`) revealed the multi-tiered gating architecture implemented by Lenovo and Intel.

### A. The Master Lock: `DPTE` in `SSDT4`

The master gatekeeper is defined in `SSDT4` under the Intel Dynamic Tuning Technology (DTT / IPF) scope:

```
Scope (\_SB)
{
    Device (IETM)
    {
        Name (DPTE, Zero)
        Method (_OSC, 4, Serialized)  // Operating System Capabilities
        {
            CreateDWordField (Arg3, Zero, STS1)
            CreateDWordField (Arg3, 0x04, CAP1)
            If ((CAP1 & One))
            {
                If ((DPTE == Zero))
                {
                    DPTE = One
                    \_SB.PC00.LPCB.EC.HKEY.DYTC (0x800F0001)
                    If (\_SB.PC00.LPCB.EC.HKEY.DHKC){}
                }
            }
            ElseIf ((DPTE == One))
            {
                DPTE = Zero
                \_SB.PC00.LPCB.EC.HKEY.DYTC (0x01FF)
                If (\_SB.PC00.LPCB.EC.HKEY.DHKC){}
            }
            ...
        }
    }
}

```

* On cold boot, `DPTE` defaults to `0`.

* In Windows, the Intel Innovation Platform Framework driver (`ipfsvc.exe`) executes `\_SB.IETM._OSC` with bit 0 of `CAP1` set to `1`.

* This handshake transitions `DPTE` from `0` to `1` and notifies the Embedded Controller via `DYTC 0x800F0001`.

* Linux does not ship with this proprietary handshake daemon, leaving `DPTE = 0`.

### B. The Gatekeeper: `DYTC` Error `0x6`

In the main DSDT (`Scope (\_SB.PC00.LPCB.EC.HKEY)`), every command attempting to switch thermal tables or fan profiles checks the `DPTE` flag:

```
If ((DPTE == Zero))
{
    Return (0x06) // Access Denied / Invalid State
}

```

Whenever the Linux kernel (`thinkpad_acpi`) attempts to set higher cooling levels, the EC responds with `0x06`, refusing to unlock higher fan RPM curves.

### C. Sensor Arbiters: `VCQL` and `DPRL`

The EC firmware tracks orientation and contact sensors to comply with IEC 62368-1 chassis touch temperature limits:

* `DPRL` (EC SRAM offset `0xC6`): Physical capacitive lap sensor.

* `VCQL`: Virtual Lap Mode flag in `DYTC`. If active (`VCQL = 1`), the cooling table clamps fan acoustic profiles and CPU TDP.

* `DYTC 0x000F1001`: Explicitly clears `VCQL`, locking the system into **Desk Mode**.

### D. The Curve Selector: `SPSC = 7`

Sending `DYTC 0x0017D001` sets sub-function `0x0D` (`SPSC = 7`, `OSML = 8`), which installs the Extreme Performance table into the ITE microcontroller and sets the internal flag `VPSC = 1`.

Reading back `DYTC 0x02` yields `0x2001f001`, where bit 13 ($0\text{x2000}$) confirms `VPSC = 1` is engaged.

## 3. The Power Profile Downgrade Bug & "Freeze Mode"

### The Issue

Once `DPTE` was unlocked (`DPTE = 1`), `thinkpad_acpi` gained full access to write to `DYTC`. When switching desktop power profiles via `power-profiles-daemon` in KDE or GNOME:

1. Setting **Power Saver** (`power-saver`) instructs the EC to load Quiet Mode (`SPSC = 1`, clamping the fan to $\sim 3200\text{ RPM}$).

2. Switching back to **Balanced** engages Adaptive Thermal Mode (**AMT**, `VAMT = 1`), which exhibits hysteresis: it retains the previous acoustic ceiling ($\sim 3200\text{ RPM}$) rather than jumping back to $3900\text{ RPM}$.

3. Only manually selecting **Performance** would restore $3900\text{ RPM}$.

### Why Windows Reboot Did Not Suffer From This

When warm-rebooting from Windows into Linux, the EC retained the active $3900\text{ RPM}$ table in SRAM, but BIOS POST reset `DPTE` back to `0`. Consequently, subsequent downgrade attempts from Linux power profile daemons were blocked with `0x06`, effectively **freezing** the $3900\text{ RPM}$ table.

### The Solution: Freeze Mode

We emulate this exact hardware behavior:

1. Set `DPTE = 1` via `_OSC` to open the gate.

2. Force Desk Mode (`DYTC 0x000F1001`).

3. Apply the full-speed cooling table (`DYTC 0x0017D001`).

4. **Relock `DPTE = 0` via `_OSC`**.

Because `DPTE` is locked back to `0`, the EC holds the $3900\text{ RPM}$ table active while blocking the OS from overwriting it. In `power-saver`, CPU frequencies and TDP limits are throttled by the kernel EPP governor anyway, keeping the laptop quiet without compromising high-load cooling.

## 4. Installation & Setup (Ubuntu 24.04 LTS)

### Prerequisites

Install `acpi-call-dkms` to allow kernel-level ACPI method execution:

```
sudo apt update
sudo apt install -y acpi-call-dkms

```

Ensure `acpi_call` loads on boot:

```
echo 'acpi_call' | sudo tee /etc/modules-load.d/acpi_call.conf

```

### Step 1: Create the Unlock Script

Create `/usr/local/sbin/lenovo-fan-unlock.sh`:

```
sudo tee /usr/local/sbin/lenovo-fan-unlock.sh << 'EOF'
#!/bin/bash
set -euo pipefail

CALL_NODE="/proc/acpi/call"

# Wait for /proc/acpi/call interface to become writable (up to 5 seconds)
for i in {1..10}; do
    if [ -w "$CALL_NODE" ]; then
        break
    fi
    modprobe acpi_call 2>/dev/null || true
    sleep 0.5
done

if [ ! -w "$CALL_NODE" ]; then
    logger -t lenovo-fan-unlock "ERROR: /proc/acpi/call is not available"
    exit 1
fi

# 1. Unlock DPTE master gate via IETM._OSC (CAP1 bit 0 = 1)
echo '\_SB.IETM._OSC b00000000000000000000000000000000 1 2 b0000000001000000' > "$CALL_NODE"

# 2. Force Desk Mode (clear VCQL lap mode flag)
echo '\_SB.PC00.LPCB.EC.HKEY.DYTC 0x000F1001' > "$CALL_NODE"

# 3. Apply full-speed cooling table (SPSC = 7, Extreme Performance, 3900+ RPM)
echo '\_SB.PC00.LPCB.EC.HKEY.DYTC 0x0017D001' > "$CALL_NODE"

# 4. Lock DPTE gate back to 0 (freeze 3900 RPM table against OS downgrades)
echo '\_SB.IETM._OSC b00000000000000000000000000000000 1 2 b0000000000000000' > "$CALL_NODE"

logger -t lenovo-fan-unlock "ThinkPad E15 Gen 4: 3900 RPM table active, DPTE gate locked"
EOF

sudo chmod +x /usr/local/sbin/lenovo-fan-unlock.sh

```

### Step 2: Create the systemd Service

Create `/etc/systemd/system/lenovo-fan-unlock.service`:

```
sudo tee  << 'EOF'
[Unit]
Description=Unlock Lenovo ThinkPad E15 Fan (DPTE & High Performance)
After=systemd-modules-load.service
Wants=systemd-modules-load.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/lenovo-fan-unlock.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

```

Enable and start the service:

```
sudo systemctl daemon-reload
sudo systemctl enable --now lenovo-fan-unlock.service

```

### Step 3: Create the Sleep / Resume Hook

The EC resets its configuration on system sleep states (S3/S0ix). Restore the table automatically on wake-up:

```
sudo tee /lib/systemd/system-sleep/lenovo-fan-resume << 'EOF'
#!/bin/sh
case "$1/$2" in
    post/*)
        /usr/local/sbin/lenovo-fan-unlock.sh
        ;;
esac
EOF

sudo chmod +x /lib/systemd/system-sleep/lenovo-fan-resume

```

## 5. Verification & Testing

### 1. Check Service Status

```
systemctl status lenovo-fan-unlock.service

```

Expected output: `Active: active (exited)` with `ThinkPad E15 Gen 4: 3900 RPM table active, DPTE gate locked` in the logs.

### 2. Check ACPI State

Verify that the `DPTE` gate is locked (`0x0`):

```
echo '\_SB.IETM.DPTE' | sudo tee /proc/acpi/call && sudo cat /proc/acpi/call
# Returns: 0x0

```

Verify that the active status register reports the Extreme Performance bit set ($0\text{x2001f001}$):

```
echo '\_SB.PC00.LPCB.EC.HKEY.DYTC 0x02' | sudo tee /proc/acpi/call && sudo cat /proc/acpi/call
# Returns: 0x2001f001

```

### 3. Load Testing

Monitor fan speed in real time:

```
watch -n 0.5 "cat /proc/acpi/ibm/fan | grep -E 'speed|level'"

```

In another terminal, generate sustained CPU load:

```
stress -c $(nproc) --timeout 45s

```

**Result:** As CPU temperature exceeds $75\text{--}80^\circ\text{C}$, fan speed smoothly climbs past the old $3200\text{ RPM}$ ceiling to $3900\text{--}3950\text{ RPM}$.

## 6. Safety & Hardware Considerations

* **Fan Bearing Lifespan:** $3900\text{ RPM}$ is the official factory top-tier speed engineered for this fan assembly and used natively by Windows under load. It does not exceed factory PWM electrical tolerances.

* **Acoustics & Idle:** In idle and low-load scenarios, the EC curve continues to scale down or turn off entirely ($0\text{--}2200\text{ RPM}$). Full speed is engaged only under thermal stress.

* **Skin Temperature:** When running heavy workloads while holding the laptop on your lap, the underside chassis may feel warmer than under stock Linux settings because higher sustained TDP is maintained.
