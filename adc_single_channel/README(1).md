# STM32F103C8T Single Channel ADC with DMA

A simple implementation of single-channel ADC reading using DMA on STM32F103C8T (Blue Pill) with LED feedback.

## Hardware Requirements

- **MCU**: STM32F103C8T (Blue Pill)
- **ADC Channel**: IN4 (PA4)
- **LED**: Connected to PB2
- **Clock**: 8MHz HSE with PLL (72MHz system clock)

## Features

- Single-channel ADC conversion using DMA
- Interrupt-driven approach with callback
- Voltage calculation (0-3.3V)
- LED toggle on each conversion (10 Hz sampling rate)
- ADC calibration on startup

## Configuration

### ADC Settings
- **Mode**: Independent mode
- **Resolution**: 12-bit (0-4095)
- **Scan Mode**: Disabled (single channel)
- **Continuous Conversion**: Enabled
- **Data Alignment**: Right alignment
- **Sampling Time**: 41.5 cycles
- **External Trigger**: Software start

### DMA Settings
- **DMA Channel**: DMA1 Channel 1
- **Direction**: Peripheral to Memory
- **Mode**: Normal (one-shot per start)
- **Priority**: High (0, 0)
- **Data Width**: Half Word (16-bit)
- **Memory Increment**: Disabled

### Clock Configuration
- **System Clock**: 72 MHz (HSE 8MHz + PLL×9)
- **ADC Clock**: PCLK2/6 = 12 MHz
- **APB1**: 36 MHz
- **APB2**: 72 MHz

## How It Works

1. **Initialization**: ADC is calibrated using `HAL_ADCEx_Calibration_Start()`
2. **Start Conversion**: DMA is started with `HAL_ADC_Start_DMA()` for single conversion
3. **Callback**: When conversion completes, `HAL_ADC_ConvCpltCallback()` is triggered
4. **Stop & Process**: DMA is stopped, `adc_ready` flag is set
5. **Main Loop**: Processes data, toggles LED, waits 100ms, restarts conversion

## Code Structure

```c
// Global variables
volatile uint16_t adc_value = 0;      // ADC raw value (0-4095)
float voltage = 0;                     // Calculated voltage
volatile uint8_t adc_ready = 0;        // Conversion complete flag

// DMA callback
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
    HAL_ADC_Stop_DMA(hadc);   // Stop after one conversion
    adc_ready = 1;             // Signal main loop
}

// Main loop
while(1)
{
    if(adc_ready)
    {
        adc_ready = 0;
        voltage = (adc_value * 3.3f) / 4095.0f;
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_2);
        HAL_Delay(100);
        HAL_ADC_Start_DMA(&hadc1, (uint32_t*)&adc_value, 1);
    }
}
```

## Pin Configuration

| Pin   | Function      | Configuration       |
|-------|---------------|---------------------|
| PA4   | ADC1_IN4      | Analog Input        |
| PB2   | LED Output    | Push-Pull, Low Speed|
| PD0   | HSE OSC_IN    | External Crystal    |
| PD1   | HSE OSC_OUT   | External Crystal    |

## Building and Flashing

### Requirements
- STM32CubeIDE or compatible toolchain
- ST-Link programmer

### Steps
1. Clone repository
2. Open project in STM32CubeIDE
3. Build project (Ctrl+B)
4. Flash to board (Run → Debug or F11)

## Usage

1. Connect analog input (0-3.3V) to PA4
2. Power the board
3. LED on PB2 will blink rapidly during startup (10 blinks)
4. LED then toggles at 10 Hz during normal operation
5. Each toggle represents a new ADC reading

## Troubleshooting

### LED Not Toggling
- Check LED polarity and current-limiting resistor
- Verify PB2 connection
- Ensure HSE crystal is working (8MHz)

### ADC Reading Incorrect
- Run ADC calibration: `HAL_ADCEx_Calibration_Start(&hadc1)`
- Check voltage reference (VDDA should be 3.3V)
- Verify PA4 input voltage is within 0-3.3V range

### HAL_Delay() Not Working
- Ensure SysTick interrupt is enabled
- Check DMA interrupt priority doesn't block SysTick
- Verify system clock is configured correctly

## Key Learnings

1. **Clock Configuration Matters**: Switching from HSI to HSE+PLL fixed timing issues
2. **Continuous Mode with Stop/Restart**: Provides better control over conversion timing
3. **Volatile Variables**: Required for variables shared between ISR and main loop
4. **DMA Callback Approach**: More CPU-efficient than polling for periodic sampling

## License

This project is provided AS-IS under STMicroelectronics software license.

## Author

Embedded Systems Engineer
Kondotty, Kerala, India

## References

- STM32F103C8T Datasheet
- STM32 HAL Driver Documentation
- RM0008 Reference Manual
