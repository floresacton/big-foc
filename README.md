# big-foc
High Power FOC Inverter

## Design Strategy

### 1. Power and Voltage Rating
- **Power Rating**: Target continuous and peak power
- **Battery Voltage**: 12S = 50.4V, 16S = 67.2V
- **Component Voltage Rating**: Components must be rated at >1.5x-2x battery voltage
  - Example: 16S battery (67.2V max) → 100V+ rated components minimum
  - Margin for voltage spikes, ringing
- **Thermal Limitation**: Max accepted power dissipation at continuous and peak power

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
  
- **Isolated Drivers**:
  - Independent high-side supply
  - Usually for high-voltage (>300V)
  - Can handle 100% duty cycle
  - Expensive, more complex

### 3. Power Loss Analysis (very important)

#### Switching Losses
- Target switching frequency from 20-25khz (non audible range & less harmonic distortion)
- Transformers and motors are inductive loads (different switching waveforms than resistive switching loss)
- Switching loss ~= 1/2*switching frequency*current*voltage*(t_on+t_off)
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
 - Good to have ~100mV at full current
 - Need amplifier right next to shunt

#### Magnetic Sensing
- Hall-effect sensing built in packages
- Galvanic isolation included
- Higher cost, useful for very high currents
- Minimal power dissipation

### 5. Thermals

#### Power Dissipation Calculation
 - On average 2 half bridges are conducting fully, one is not conducting
- total switching loss ~= 4*single switch loss
- gate charge loss is usually quite small compared to switching and resistive losses for 25khz
- total conduction loss ~= 2*single conduction loss

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
4. Junction temp = Ambient + (Total losses * Total thermal resistance)
5. Target <100C but <60C is better

### 6. Protection and Sensing

#### Temperature Monitoring
- NTC thermistor on heatsink or FETs

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
 - di/dt * L = voltage induced over drain source
 - Low esr (usually MLCC) caps in low inductance switching loop
 - very rough calc Q_sustain = t_deadtime*I_peak
 - Q_sustain / V_spike = C_switch loop
 - Goal to hit V_spike <1% of V_bus
 - Keep loops as small as possible
 - Shunt amplifier right next to shunt resistor

### 10. Example Calculations (for my inverter)
 - 

