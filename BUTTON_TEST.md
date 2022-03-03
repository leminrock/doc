## test buttons on every pin

| status     | pin | pin | status     |
| ---------- | --- | --- | ---------- |
| +3.3V      | 1   | 2   | +5.0V      |
| **best**   | 3   | 4   | +5.0V      |
| **best**   | 5   | 6   | GND        |
| **no**     | 7   | 8   | **best**   |
| GND        | 9   | 10  | **best**   |
| **PULLUP** | 11  | 12  | **no**     |
| **PULLUP** | 13  | 14  | GND        |
| **PULLUP** | 15  | 16  | **no**     |
| +3.3V      | 17  | 18  | **PULLUP** |
| **best**   | 19  | 20  | GND        |
| **PULLUP** | 21  | 22  | **no**     |
| **best**   | 23  | 24  | **best**   |
| GND        | 25  | 26  | ADC0       |