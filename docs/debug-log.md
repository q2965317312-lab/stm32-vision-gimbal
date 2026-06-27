# Debug Log

## OLED Test

Status: Passed

- Verified 5V input and 3.3V output.
- OLED displayed test text correctly.
- Confirmed STM32 firmware was running normally.

## Key Test

Status: Passed

- KEY1 to KEY4 were tested.
- OLED showed correct key values.

## Stepper Motor Test

Status: Passed

- Yaw and Pitch motors were tested separately.
- STEP, DIR and EN signals were verified.
- Both axes could rotate in forward and reverse directions.

## TMC2209 Issue

Status: Fixed by replacing module

Problem:
- One motor had no holding torque.
- One TMC2209 module became hot even without motor connected.

Debug steps:
- Checked VM, VIO and EN voltage.
- Swapped TMC2209 modules between Yaw and Pitch.
- The issue followed the TMC2209 module.

Conclusion:
- The TMC2209 module was faulty.
