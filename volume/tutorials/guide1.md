#### LB & LED pulser
 
The master FPGA on the readout board is able to generate `feedback pulses` with controlled length, rate, and phase in order to perform FEE and TDC calibration. Two types of feedback pulses are used:
 
- **`LED pulses`**: Forwarded to individual channel LEDs. Allows data collection via LED light. The pulse length defines the SiPM signal amplitude, the pulse rate defines the collected hit rate, and the phase doesn't matter. `LED pulses` are generated exclusively by the FPGA.
- **`LB pulses`** (loopback pulses): Forwarded to individual TDC channels. There is a set of switches (SW, 5 in proto v2) that switch groups of channels between FEE and the LB pulser. Thus, TDC data can be sourced from precise signals with controlled form. In turn, a set of switch groups (LB, 2 in proto v2) can be sourced from either the FPGA LB pulser or an external generator. The switches and loopback source are controlled by registers `SW[4..0]` and `LB[1..0]` (see [control register](#Control registers)). The SW and LB switch-channel mapping can be found in [Channels mapping](#channels-mapping).
 
Both `LED` and `LB` pulses are generated using the same FPGA logic and share controlled parameters:
 
- The pulse rate, controlled by the 24-bit register `pls period`, is multiplied by 320 ns, providing a range of **3.125 MHz to 0.19 Hz**.
- The FPGA pulser generator uses an MMCM (mixed-mode clock manager) for clock source, phase, and fine-controlled pulse length.
 
For `LB pulses`:
 
- In **asynchronous clock mode** (`pls clk async = 0`), the pulse length is fixed at a single clock cycle (32 ns).
- In **synchronous mode**, the pulse length is the same for both `LB` and `LED` pulses and is fine-tuned using a sourced and phase-shifted clock.
 
### Pulse Control Registers:
 
- **`pls len`**: Controls the pulse length in 32 ns clock cycles.
- **`pls phase`**: Controls the MMCM clock phase shift in 12.987 ps increments.
- **`pls use OR`**: Determines whether the fine-tuned pulse is formed by an `OR` or `AND` operation between the original and shifted pulses. This avoids metastability and allows pulse lengths in the range of **~1.5 ns to 8.16 µs** with step 13 ps.
 
### Additional Registers:
 
- **`pls reset`**: Resets the MMCM (must be applied only after switching between `sync`/`async` modes).
- **`pls ready`** (status register): Indicates whether the pulser is ready after parameter changes.
 
<details>
<summary>Pulse control code example</summary>
 
Well tested C code example to FPGA pulser control operation.
 
```cpp
void setup_pulse(ipbus_connection &slow_control, bool pulser_async, double pulse_freq = 10./*kHz*/, double pulse_lenght = 32./*ns*/, double mmcm_phase = -1/*0..1*/) {
    const float mmcm_freq = 31.25; // MHz
    const int period_cnt = 10;
    const int mmcm_ph_steps = 0x99f;
    const float mmcm_phase_metamin = 90./mmcm_ph_steps; // eq 1.17 ns
 
    double plen_periods, plen_phase;  
    bool plen_use_or = false;
    plen_phase = std::modf(pulse_lenght*mmcm_freq/1e3, &plen_periods);
    plen_periods += 1;
 
    // avoiding metastability 
    if(plen_phase < mmcm_phase_metamin)
    {
        // too small pulse
        if(plen_periods < 2)
        {
            plen_phase = mmcm_phase_metamin;
            plen_use_or = false;
        }
        else
        {
            plen_phase = 1-plen_phase;
            plen_periods -= 1;
            plen_use_or = true;
        }
    }
 
    uint32_t freq_reg = ceil(mmcm_freq/pulse_freq*1e3/period_cnt);
    uint32_t plen_reg = plen_periods;
    uint32_t phase_reg = (mmcm_phase<0.)? ceil(plen_phase*mmcm_ph_steps) : ceil(mmcm_phase*mmcm_ph_steps);
    printf("pulser set: Async %i; len %.2f ns; rate: %.1f kHz\n", pulser_async, pulse_lenght, pulse_freq);
    printf("pulser param: plen: %i; periods: %i; %s_phase: %.3f (%i); OR: %i\n",
        plen_reg, freq_reg, (mmcm_phase<0.)?"plen":"mmcm", (mmcm_phase<0.)?plen_phase:mmcm_phase, phase_reg, plen_use_or);
 
    write_bit(slow_control, plen_use_or, 2, 0x1);
    write_bit(slow_control, pulser_async, 3, 0x1); // false = sync
    slow_control.write_single((plen_reg<<24) + freq_reg, 0x6); // pulser len, period
    slow_control.write_single(phase_reg, 0x7); // pulser phase
 
    // reset needed only after switch sync/async else reset must not be applyed
    bool is_pulser_async = slow_control.read_single(0x1) & (1<<3);
    bool mmcm_reset = is_pulser_async != pulser_async; 
    if(mmcm_reset)
    {
        // reset pulser
        write_bit(slow_control, true, 2, 0x0); // pulser sync
        write_bit(slow_control, false, 2, 0x0); // pulser sync
        printf("pulser reset...\n");
    }
 
    // wait ready
    while(true)
    {
        bool pls_ready = slow_control.read_single(0x102) & (1<<26);
        printf("pulser ready: %i\n", pls_ready);
        if(pls_ready) break;
    }
 
    return;
};
```
 
</details>
