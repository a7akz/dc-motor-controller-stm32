# STM32 DC Motor Controller — Project Log

## Project aim

Design and test a PWM-controlled brushed DC motor controller using an STM32 NUCLEO-G474RE, an external MOSFET power stage, flyback protection, and measured verification.

Main goals:
- Generate PWM from STM32.
- Use MOSFET as a low-side switch.
- Drive a DC motor from an external bench supply.
- Verify each stage through measurements.
- Document design decisions, test results, and faults.

## Stage 1 — STM32 GPIO bring-up


Goal:
Confirm that the NUCLEO-G474RE board, STM32CubeIDE project, flashing process, GPIO output, and basic delay timing work.

Setup:
- Board: NUCLEO-G474RE
- IDE: STM32CubeIDE
- Output: onboard LED

Test:
The onboard LED was controlled using GPIO output and delay timing.

Result:
- LED turned on for 0.5 seconds.
- LED turned off for 2 seconds.
- Repeated blink pattern worked correctly.

Conclusion:
STM32CubeIDE project creation, flashing, GPIO output, and basic timing were verified.

## Stage 2 — Serial output test

Goal:
Confirm that the STM32 can send serial debug output to the PC.

Setup:
- Board: NUCLEO-G474RE
- Serial interface:
- PC terminal software:
- Baud rate:

Test:
The STM32 transmitted text to the PC terminal.

Result:
- Serial output was received successfully on the PC.

Conclusion:
Serial debugging is working and can be used later for duty cycle/state logging.

## Stage 3 — PWM gate signal test

Date: 22/06/2026

Goal:
Verify that the STM32 PWM output reaches the MOSFET gate correctly before connecting an external load.

Setup:
- PWM pin:
- Timer/channel:
- Gate resistor:
- Pulldown resistor:
- MOSFET:
- Measurement point: MOSFET gate-to-source voltage

Measurements:

| Duty cycle | Measured average gate voltage |
|---:|---:|
| 25% | 0.81 V |
| 50% | 1.628 V |
| 75% | 2.44 V |
| 100% | 3.25 V |

Expected behaviour:
Average gate voltage should approximately equal duty cycle × 3.25 V.

Conclusion:
PWM generation and gate signal delivery were verified. The measured values match the expected average PWM voltage.

## Stage 4 — MOSFET resistor/LED load test

Date: 24 June 2026

Goal:  
Verify that the STM32 PWM signal can switch an external LED/resistor load through the IRLB8721 MOSFET.

Circuit/setup:
- Board: NUCLEO-G474RE
- Main components: IRLB8721 MOSFET, LED, 220 Ω load resistor, 220 Ω gate resistor, 10 kΩ gate pulldown
- Supply: 5 V from bench PSU, current limit 0.2 A
- Key pin/peripheral: PB3 / TIM2_CH2 PWM
- Important wiring note: STM32 GND, PSU negative, MOSFET source, and the bottom of the 10 kΩ pulldown are connected to the same common ground.

Circuit diagram:

```text
       +5 V from PSU
            │
            │
         LED anode
            │
         LED cathode
            │
       220 Ω load resistor
            │
            │
       Drain — IRLB8721 — Source
                              │
                              │
PSU negative / common ground ─┴──────── STM32 GND
                              │
                              │
                          10 kΩ pulldown
                              │
PB3 PWM / TIM2_CH2 ─ 220 Ω ───┘
                         gate resistor
```

Expected result:
- 0% duty: LED off.
- Increasing duty cycle: LED becomes brighter.
- 100% duty: LED fully on.
- Gate PWM switches between approximately 0 V and 3.25 V.
- MOSFET remains cool.

Measurements / observations:

| Check | Result |
|---|---|
| PSU setting | 5 V, current limit 0.2 A |
| Common ground | STM32 GND connected to PSU negative |
| 0% duty | LED off |
| 75% duty | Gate PWM measured at approximately 0–3.25 V, around 9.9 kHz, LED bright |
| 100% duty | LED fully on |
| MOSFET temperature | Remained cool |
| Drain waveform | Drain switched opposite to gate waveform |

Evidence captured:
- Wiring photo: [Stage 4 Wiring](../photos/Stage%204%20Evidence/Stage%204%20Wiring.jpeg)
- Oscilloscope screenshot: [Stage 4 waveform](../photos/Stage%204%20Evidence/Stage%204%20waveform.png)
- Waveform CSV: [Stage 4 waveform CSV](../photos/Stage%204%20Evidence/Stage%204%20waveform.csv)
- PSU photo: [PSU photo](../photos/Stage%204%20Evidence/PSU%20photo.jpeg)

Problems found:

Before the common ground/reference was correctly understood, the circuit behaved unpredictably. The LED could turn on/off when measurement leads or the oscilloscope ground were connected.

Likely cause:  
STM32 GND and PSU negative were not initially being treated as the same reference node. Since the MOSFET is controlled by VGS, the STM32 PWM signal must share the same ground reference as the MOSFET source.

Fixes made:

STM32 GND was connected to PSU negative, making STM32 GND, PSU negative, MOSFET source, and the gate pulldown reference the same electrical node.

Conclusion:

Stage 4 passed. The STM32 PWM signal successfully drove the IRLB8721 gate, and the MOSFET switched the external LED/resistor load. The LED brightness changed with duty cycle, the gate waveform was approximately 0–3.25 V at around 9.9 kHz, and the MOSFET remained cool. This confirmed that the low-side MOSFET switching circuit works before moving to a motor load.

Next step:

Move to Stage 5: open-loop DC motor PWM test with a flyback diode and supply capacitor.

## Stage 5 — Open-loop DC motor PWM test

Date: 25/06/2026

Goal:  
Verify that the STM32 PWM signal can control a DC motor through the IRLB8721 MOSFET with flyback protection fitted.

Circuit/setup:
- Board: NUCLEO-G474RE
- Main components: IRLB8721 MOSFET, DC motor, 1N5822 flyback diode, 220 Ω gate resistor, 10 kΩ gate pulldown, bulk capacitor
- Supply: 5V PSU
- Current limit: 0.2A
- Key pin/peripheral: PC0 / configured timer PWM channel
- Important wiring note: STM32 GND, PSU negative, MOSFET source, and the bottom of the 10 kΩ pulldown are connected to the same common ground.

Circuit diagram:

```text
       +V from PSU
            │
            ├───────────────┐
            │               │
          Motor          1N5822 diode
            │        cathode/stripe ↑
            │        anode ↓
            ├───────────────┘
            │
       Drain — IRLB8721 — Source
                              │
                              │
PSU negative / common ground ─┴──────── STM32 GND
                              │
                              │
                          10 kΩ pulldown
                              │
PC0 PWM ─ 220 Ω ──────────────┘
        gate resistor

Bulk capacitor:
capacitor + → PSU +
capacitor - → PSU -
```

Expected result:
- 0% duty: motor off.
- Increasing duty cycle: motor speed increases.
- 100% duty: motor runs at maximum speed for the chosen supply voltage.
- Flyback diode protects the MOSFET from motor inductive kickback.
- MOSFET remains cool or only slightly warm.
- STM32 does not reset or behave unpredictably.

Measurements / observations:

| Check | What to record | Result |
|---|---|---|
| PSU setting | Supply voltage and current limit | 5 V, 0.2 A |
| 0% duty | Whether motor is fully off | Off |
| Low duty | Duty percentage and whether motor starts spinning | 25% duty, motor starts spinning |
| Medium duty | Duty percentage and whether motor speed clearly increases | 50% duty, clear increase in speed |
| 100% duty | Whether motor reaches maximum speed for chosen supply voltage | 100% duty, motor reaches maximum speed for 5 V supply|
| Gate waveform | PC0 gate signal voltage/frequency | Approximately 0–3.32 V PWM at around 9.9 kHz |
| PSU current | Current shown on PSU during motor running | Approximately 0.01 A while running at 25% duty and 0.131A at 100% duty |
| MOSFET temperature | Cool, slightly warm, hot | Slightly warm |
| Drain waveform | Any obvious large spike/ringing when PWM switches | Very small switching spike observed, peak below 5.68 V; no excessive drain spike seen|
| STM32 behaviour | Whether board stays stable or resets/glitches | Stable, no resets or glitches |

### Stage 5 wiring

![Stage 5 wiring](../photos/Stage%205%20Evidence/Stage%205%20Wiring.jpeg)

### Motor waveform at 100% duty

![Motor waveform at 100% duty](../photos/Stage%205%20Evidence/Motor%20Waveform%20100%25%20duty.png)

### PSU setup

![DC PSU](../photos/Stage%205%20Evidence/DC%20PSU.jpeg)

Problems found:

No major faults were found. A very small switching spike was observed on the drain waveform, but the peak remained below approximately 5.68 V, which is well below the IRLB8721 VDS limit of 30 V.

Fixes made:

Flyback diode and bulk capacitor were fitted before testing the motor. The common ground connection from Stage 4 was maintained.

Conclusion:

Stage 5 passed. The STM32 PWM signal on PC0 successfully controlled the DC motor through the IRLB8721 MOSFET. Increasing duty cycle increased motor speed, the STM32 remained stable, and the drain switching spike remained small with the flyback diode fitted. The MOSFET became only slightly warm during testing.

Next step:

Move to improving the control method, such as serial/button/potentiometer duty control, before later adding RPM sensing.