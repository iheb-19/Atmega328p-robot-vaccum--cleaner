# ATmega328P Robot Vacuum Cleaner Controller

Embedded C application for the ATmega328P microcontroller that simulates a robot vacuum cleaner with 5-button sequence authentication, 5-second inactivity timeout, and an indefinitely alternating 2-phase operation cycle (vacuuming + movement) until manually stopped.

---

## Features

- **5-button sequence authentication** (B2 → B3 → B2 → B3 → B2) to start the robot
- **5-second inactivity timeout** at every step — sequence resets if no input received
- **Progressive LED feedback** using binary-style PORTC write at each step
- **Wrong sequence**: all LEDs off, silent reset to PRIMO
- **Indefinitely alternating cycle**: Vacuuming (4s) ↔ Movement (2s) until B5 pressed
- **Distinct PWM profiles** per phase on separate LEDs (L5 for vacuum, L4 for movement)
- **8-state FSM**: PRIMO → SECONDO → TERZO → QUARTO → QUINTO → VERIFICA → ASPIRAZIONE → SPOSTAMENTO

---

## Authentication Sequence

```
B2 → B3 → B2 → B3 → B2
```

- Timer1 starts on first button press for timeout tracking
- Timeout (ciclo == 10 at 0.5s/tick = 5s): sequence resets to PRIMO, all LEDs off
- Wrong sequence: `vero` counter below 5 → VERIFICA resets to PRIMO silently

---

## Operating Phases

| Phase        | Duration | LED | Period | Duty Cycle |
|--------------|----------|-----|--------|------------|
| ASPIRAZIONE  | 4s       | L5  | 300ms  | 33%        |
| SPOSTAMENTO  | 2s       | L4  | 500ms  | 20%        |

Phases alternate indefinitely. B5 stops the robot from either phase and resets to PRIMO.

---

## Hardware Mapping

| Pin          | Function                              |
|--------------|---------------------------------------|
| PORTD2 (B2)  | Auth steps 1, 3, 5                    |
| PORTD3 (B3)  | Auth steps 2, 4                       |
| PORTD5 (B5)  | Stop robot                            |
| PORTC0 (L0)  | Auth progress indicator               |
| PORTC1 (L1)  | Auth progress indicator               |
| PORTC2 (L2)  | Auth progress indicator               |
| PORTC3 (L3)  | Correct sequence confirmed            |
| PORTC4 (L4)  | Movement simulation (PWM)             |
| PORTC5 (L5)  | Vacuum simulation (PWM)               |

---

## Timer Configuration

| Timer  | Role                                  | OCR Value | Interval |
|--------|---------------------------------------|-----------|----------|
| Timer0 | PWM tick (L4 + L5)                   | 77        | 5 ms     |
| Timer1 | Phase duration + inactivity timeout   | 7812      | 500 ms   |

Timer1 starts on first button press. Timer0 starts after successful authentication. Both stop on B5 reset.

---

## State Machine

```
PRIMO ──[any key]──► SECONDO ──[any key / timeout]──► TERZO ──[any key / timeout]──►
QUARTO ──[any key / timeout]──► QUINTO ──[any key / timeout]──► VERIFICA
                                                                  /        \
                                                           [vero==5]    [vero<5]
                                                               │              │
                                                          ASPIRAZIONE      PRIMO
                                                               │  ▲
                                                           [4s / B5]
                                                               │
                                                         SPOSTAMENTO
                                                               │
                                                           [2s / B5]
                                                               │
                                                          ASPIRAZIONE (loop)
```

---

## PWM Implementation

All PWM runs inside `ISR(TIMER0_COMPA_vect)` (5ms tick):

| State        | LED | ON ticks | Period ticks | Duty cycle |
|--------------|-----|----------|--------------|------------|
| ASPIRAZIONE  | L5  | 20       | 60           | 33%        |
| SPOSTAMENTO  | L4  | 20       | 100          | 20%        |

Each phase uses a dedicated LED — no shared PWM channel between phases.

---

## Technical Highlights

- Bare-metal C, no Arduino libraries
- Alternating 2-phase infinite loop — common pattern in autonomous embedded systems
- Per-step timeout watchdog using Timer1 ciclo counter
- PORTC written directly with binary value (0x1–0x5) for compact LED progress display
- Independent PWM per phase on separate output pins
- B5 stop handled identically in both active states for clean symmetric design
- PCINT2 interrupt with edge detection for all buttons
- Volatile variables for safe ISR ↔ main loop communication

---

## Build

Compiled with AVR-GCC for ATmega328P (16 MHz clock).

```bash
avr-gcc -mmcu=atmega328p -DF_CPU=16000000UL -O2 -o vacuum.elf main.c
avr-objcopy -O ihex vacuum.elf vacuum.hex
avrdude -c usbasp -p m328p -U flash:w:vacuum.hex
```

---

## Author

Ehab Guezmir — B.Sc. Computer Engineering and Automation, eCampus University
