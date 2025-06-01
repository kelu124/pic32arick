# pic32arick


## Capacities

Why not use the PIC32AK1216GC41064 ? It has 40Msps ADC and two OPAMPS.

* 128kB Flash
* 16kB RAM
* Three Rail-to-Rail 100 MHz Operational Amplifiers 
* Three 4-Wire SPI Modules 
* 3x 12bits DAC
* 2x 40Msps ADC, 12bits
* High res PWM, 2.5ns res

## what do we have

* a 5V pulser (add a 5V booster pin as trigger)
* a couple of opamps to amplify the received signal
* UART to communicate
* i2c to manage components (R)

## Design

* ADD a MD0100 to protect ADC
* Add a 5V -> 3.3V LDO to supply chip
* Add a 3.3 -> 5V trigger (for pulser)
  * Also add 3.3V pulser to inputs to comparator to add trigger to ADC
* MUX for input post MD0100:
  * either MD0100 input
  * or DAC to ADC
* i2C R for opamp gain 
* i2c (different) to SAO
* add a tx/rx swapper (2xR0)

* Use RaspberryPi header with 5V, 3.3V, I2C, UART (5 first pins)
* Also add SAO 1.69bis standard header for I2C + UART




## Others

* [Programmer refs](https://ww1.microchip.com/downloads/aemDocuments/documents/MCU16/ProductDocuments/ReferenceManuals/PIC32A-Programmers-Reference-Manual-DS70005610.pdf)
* [Hardware datasheet](https://ww1.microchip.com/downloads/aemDocuments/documents/MCU16/ProductDocuments/DataSheets/PIC32AK1216GC41064-Family-Data-Sheet-DS70005592.pdf)



## Resources

* [MPLab Snap](https://www.microchip.com/en-us/development-tool/pg164100) as a debugger at 10e or on [mouser](https://www.mouser.fr/ProductDetail/Microchip-Technology/PG164100?qs=w%2Fv1CP2dgqoaLDDBjfzhMQ%3D%3D&mgh=1&vip=1&srsltid=AfmBOorfcviOZGzz0DrBrRMUarEe0MIloAbGO8anevupSmwxqNnA8rHNjHU).
* [Curiosity Platform Development Board and PIC32AK1216GC41064 GP DIM Bundle](https://www.mouser.fr/ProductDetail/Microchip-Technology/BN61G23A?qs=jcD%2FCkGBYeMEshRNnSisEQ%3D%3D&srsltid=AfmBOoop7UIh4PZcPw9w2F92mV66UVmYFCnYubT6DvKy7wVqab2wA-UX) (or BN61G23A)
