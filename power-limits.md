# Intel RAPL Power Limits Tuning on ThinkPad E15 Gen 4 (Intel i7-1255U)

## Why Adjust Power Limits?

With the fan unlocked to 3900+ RPM, the cooling system can sustain significantly more heat than the default firmware profile allows:
* **Default behavior:** Aggressive burst up to 55W (causing instant 95°C+ spikes), followed by a conservative drop to ~20–25W.
* **Tuned behavior (PL1 = 32W / PL2 = 45W):** Eliminates thermal shock while sustaining ~3.4–3.6 GHz across all 12 threads under heavy multi-core load (compilation, rendering), stabilizing temperatures at **83–86°C**.

## Why Not Use `throttled` (lenovo-throttling-fix)?

On modern Linux distributions (such as Ubuntu 24.04 LTS) with **Secure Boot enabled**, the kernel operates in **Lockdown Mode (`integrity`)**:
* Direct access to CPU MSRs (`/dev/cpu/*/msr`) and physical memory (`/dev/mem`) is blocked by the kernel.
* `throttled` fails to write power limits and TCC offsets unless Secure Boot / Kernel Lockdown is disabled.
* 12th Gen Intel CPUs (Alder Lake) have hardware-locked voltage planes (FIVR Lock), making undervolting impossible regardless of tools used.

## The Clean Solution: Native Kernel `intel_rapl`

The Linux kernel provides the native **Power Capping Framework (`intel_rapl`)** at `/sys/class/powercap/intel-rapl/`:
* Writes through this interface are handled in kernel space and are **fully compatible with Secure Boot and Kernel Lockdown**.
* No background daemon or memory overhead required.
* Settings persist across AC disconnects; on battery, the embedded controller (EC) automatically clamps power at hardware level for safety.

## Configuration Reference

In `/usr/local/sbin/lenovo-fan-unlock.sh`:

| Parameter | Recommended Value | Description |
|---|---|---|
| `ENABLE_RAPL_TUNING` | `1` | Enables power capping injection |
| `PL1_WATTS` | `32` | Long-term sustained power limit (W) |
| `PL2_WATTS` | `45` | Short-term burst power limit (W) |

Verify active constraints via sysfs:
```bash
cat /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_power_limit_uw  # PL1
cat /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_1_power_limit_uw  # PL2
