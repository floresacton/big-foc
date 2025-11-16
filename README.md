# big-foc
Field Oriented Control Inverter

70V 160A 10kW continuous

12kW peak

99% efficiency at 8kW

48$ BOM Cost

65x75x32mm size

Ignoring the heatsink, that's 64kW/L :)

## Design Strategy

### 1. Power and Voltage Rating
- **Power Rating**: Target continuous and peak power
- **Battery Voltage**: 12S = 50.4V, 16S = 67.2V
- **Component Voltage Rating**: Components must be rated at >1.5x-2x battery voltage
  - Example: 16S battery (67.2V max) → 100V+ rated components minimum
  - Margin for voltage spikes, ringing
- **Thermal Limitation**: Max accepted power dissipation at continuous and peak power, <=100C

### 2. Power Stage Design

#### Switch Selection by Voltage
- **Low Voltage (<300V)**: MOSFETs
  - Aim for lower RDS(on) better conduction losses
  - Aim for lower total gate charge for better switching losses
  - Most cost-effective (100V rated components very common)
- **High Voltage (>300V)**: SiC FETs or IGBTs
  - SiC FETs: Fast switching, lower losses
  - IGBTs: Slower switching, higher switching losses, very high power

#### Gate Driver Selection
- **Bootstrap Drivers**: 
  - Simple, low cost
  - Requires minimum duty cycle for bootstrap to charge
  - Usually fine until back emf approaches battery voltage
  - Pick a fast reverse recovery charge diode
  
- **Isolated Drivers**:
  - Independent high-side supply
  - Usually for high-voltage (>300V)
  - Can handle 100% duty cycle
  - Expensive, more complex

### 3. Power Loss Analysis (very important)

#### Switching Losses
- Target switching frequency from 20-25khz (non audible range & less harmonic distortion)
- Transformers and motors are inductive loads (different switching waveforms than resistive switching loss)
- Switching loss ~= 1/2\*switching frequency\*current*voltage\*(t_on+t_off)
- t_on, t_off ~= Q_gate/I_charge
- I_charge = gate drive current

#### Conduction Losses
- I^2*R × rds_on (for mosfets)
- rds_on increases with temperature so we can use parallel FETs without runaway (thermals matter too)

### 4. Current Sensing

#### Sensing Approach
- **Phase count**:
  - Measure 2 phases calculate third (more noise)
  - Measure 3 phases and average (less noise, better if fits & symmetry)

- **In-Phase**:
  - Able to measures phase currents regardless of switch state
  - Higher cost, a lot of common mode noise
  - Better current measurements if done properly
  
- **Low-Side (1-3 shunts)**:
  - Cheaper
  - Simpler ground referenced amplifier design
  - Can only sample during low side conduction
  - Use stm32 timer ADC synchronization

#### Shunt Resistors
 - Should be around or slightly below rds_on of FETs
 - Good to have about 50-75mV at full current
 - Select for 20V/V amplifier (INA181A1 super common & cheap)
 - <=50ppm for 100C temp rise scenario
 - Need amplifier right next to shunt
 - Need center VREF amp for negative and positive current
 - Need to choose settling time to 1% better than 95% duty cycle (bootstrap is limiting factor)

#### Magnetic Sensing
- Hall-effect sensing built in packages
- Galvanic isolation included
- Higher cost, useful for very high currents
- Minimal power dissipation

### 5. Thermals

#### Power Dissipation Calculation
 - On average 2 half bridges are conducting fully, one is not conducting
- total switching loss ~= 4\*single switch loss
- gate charge loss is usually quite small compared to switching and resistive losses for 25khz
- total conduction loss ~= 2\*single conduction loss

#### FET Cooling Orientation
- **Top-Side Cooling**: FETs mounted with drain pad facing up
  - More expensive
  - Easier to cool and route
  
- **Bottom-Side Cooling**: FETs with dissipation through PCB
  - Cheaper
  - Easier assembly
  - Harder low inductance loop

#### Thermal Design Process
1. Calculate junction-to-case thermal resistance (datasheet)
2. Add PCB thermal resistance
3. Add heatsink thermal resistance
4. Junction temp = Ambient + (Total losses \* Total thermal resistance)
5. Target <100C but <60C is better

### 6. Protection and Sensing

#### Temperature Monitoring
- NTC thermistor (cheaper than PTC) on heatsink or FETs
- Lower B kind of better like 3400 because more linear over big range
- 10k 3400B ~= 1k at 100C, 30k at 0C

#### Fast Overvoltage Protection
- Hardware comparator for <1µs response time
- Trigger emergency PWM shutdown (stm32 timer break input)
- Threshold = 0.9% or something of voltage rating
- Directly disables gate drivers
- Optional avalanche diode for last resort protection

#### Voltage Measurement
- Resistor divider to ADC with a little bit of filtering
- Per-phase voltage for locking onto back emf for high speed

### 7. Microcontroller Selection

#### STM32G4 Series
**Key Features for FOC:**
- High-resolution timers with dead-time insertion and center aligned PWM
- Fast 12-bit ADCs
- ADC triggering synchronized to PWM for current sampling
- Multiple ADCs for simultaneous phase current sampling
- Math acceleration (FPU, CORDIC) for FOC
- Sufficient for >20kHz control loop

**Typical MCU Choices:**
- STM32G431: Small packages, cost-optimized
- STM32G474: More peripherals, flash, and RAM for complex algorithms

### 8. FOC Control Implementation

#### Control Loop Structure
- Current loop: 10-50kHz (1/10 of switching frequency typical)
- Velocity loop: 1-5kHz
- Position loop: 100-1000Hz

#### Key Considerations
- ADC sampling synchronized to PWM valley/center
- Current reconstruction for low-side shunts
- PI controller tuning based on motor parameters
- Clarke/Park transforms for coordinate conversion
- Space Vector Modulation (SVM) for efficient bus utilization
- Add third harmonic for higher effective bus voltage

### 9. PCB Layout (Very very important)
 - Low inductance switching loop
 - di/dt \* L = voltage induced over drain source
 - Low esr (usually MLCC) caps in low inductance switching loop
 - very rough calc Q_sustain = t_deadtime*I_peak
 - Q_sustain / V_spike = C_switch loop
 - Goal to hit V_spike <1% of V_bus
 - Keep loops as small as possible
 - Shunt amplifier right next to shunt resistor

### 10. Connectors
 - Waterproof, either SD13 series or DT Duetch (orange ones) are common
 - Phase and power wires just shrouded, bolt internally

### 11. Example Calculations (for my inverter)
- **Motor Characteristics**
	- 40uH phase to phase inductance
	- AMKs have 2.5nF inter winding capacitance
	- Couldn't find inter-winding capacitance of mine, but I think its relatively small losses compared to conduction
- [MOSFET loss equations](https://www.ti.com/lit/an/slyt664/slyt664.pdf?ts=1762657470589)
- [Layout and more equations](https://www.infineon.com/assets/row/public/documents/24/42/infineon-designing-an-optimized-half-bridge-dc-dc-converter-with-medium-voltage-coolgan-transistors-applicationnotes-en.pdf)
 - **Losses**
	 - Power 8.4kW at 120A rms current or two phases at 170A
	 - 2 phases on at a time on average (same thing as 3 RMS currents)
	 - parallel rds @ 15V: 0.55mohm at 25C, 0.85mohm at 100C
	 - P_cond_tot = 2\*170\*170\*0.00055 = 32W
	 - P_shunt = 2\*170\*170\*0.00033 = 19W
	 - Q_gs2 = 20nC, Q_gd = 50nC, Q_eff = 70nC
	 - Q_actual = 140nC cause parallel FETs
	 - P_sw = 2\*70V\*170A\*25khz\*140nC/4A = 20.8W
	 - P_sw_total = 8.925W\*2 half bridge = 41.6W
	 - P_coss = 12FETs \* 25khz \* 70V^2 \* 2.3nF = 3.4W
	 - P_winding_cap = 3 phase \* 25khz \* 70V^2 \* 2.5nF = 0.9W
	 - P_gate_total = 12 FETs\*250nC\*15V\*25khz = 1.1W
	 - P_gate_drive = <1W
	 - Deadtime 150ns
	 - P_diode = ~1.5\*1.2V\*170A\*25khz\*150ns = 1.2W (a bit sus math but reasonable)
	 - 4% dissipation factor for 1206 2.2uF ~ 10mohm I think for equivalent 10 (randomish number)
	 - Sw_energy = 70V\*170A\*140nC/4A = 0.4mJ per phase
	 - C_droop ~= 0.4mJ/(20uF \* 70V) = 0.3V
	 - The Vrms of the droop is prob below ~50mV
	 - P_cap_sw = 50mV^2/10mohm = 0.25W*3phases comes to <1W
	 - I_ripple ~= 110A
	 - Assuming 70A into ceramics and 40A into electrolytic
	 - 80 ceramics / 70A = 0.87A per ceramic
	 - P_ripple_cer = 0.875^2 \* 100mohm = 80mW
	 - P_cer_tot = 0.08*80 = 6.4W
	 - Electrolytic 3mohm at 25khz (susish think ok)
	 - I_elec ~= 40A/10 = 4A
	 - P_elec = 4A^2 \* 3mohm = 48mW
	 - P_elec_tot = 48mW \* 10 = 0.5W
	 - Total at 8.4kW = 32+41.6+1.1+1.2+1+1+6.4+0.5+3.4+19= 107.1W
	 - About 98.8% efficiency
- **Control**
	- Max phase current 220A, 210A max (170rms) is 12kW
	- SVPWM
	- Optional 3rd harmonic overmodulation
	- I_phase_rms = I_phase_pk / sqrt(12)
	- I_pk_pk_ripple = 70V / (2 \* 40uH \* 25khz) \* D \* (1-D) = 8.75A ; D=0.5
	- I_pk = 4.4A
	- I_rms = I_pk_pk_ripple / sqrt(12) = 2.5A
