# STM32 Single Channel ADC with DMA

A complete guide for configuring and using a single-channel ADC with DMA in STM32CubeIDE.

## Table of Contents
- [Overview](#overview)
- [Hardware Requirements](#hardware-requirements)
- [Software Requirements](#software-requirements)
- [Configuration Steps](#configuration-steps)
- [Implementation](#implementation)
- [Usage Examples](#usage-examples)
- [Troubleshooting](#troubleshooting)
- [Additional Resources](#additional-resources)

## Overview

This project demonstrates how to set up and use the STM32 ADC (Analog-to-Digital Converter) with DMA (Direct Memory Access) for efficient, CPU-independent analog signal acquisition. Using DMA eliminates the need for CPU intervention during each conversion, making it ideal for continuous monitoring applications.

### Features
- Single-channel ADC configuration
- DMA-based automatic data transfer
- Continuous conversion mode
- Minimal CPU overhead
- Real-time voltage monitoring

## Hardware Requirements

- STM32 Development Board (e.g., STM32F4 Discovery, Nucleo boards)
- Analog input source (sensor, potentiometer, etc.)
- USB cable for programming and debugging

### Pin Configuration
The ADC channel pin will be automatically assigned by STM32CubeMX based on your microcontroller variant. Common pins include:
- **STM32F4**: PA0 (ADC1_IN0), PA1 (ADC1_IN1), etc.
- **STM32L4**: PA0 (ADC1_IN5), PA1 (ADC1_IN6), etc.

**Note**: Ensure your input voltage does not exceed VREF+ (typically 3.3V).

## Software Requirements

- **STM32CubeIDE** (latest version recommended)
- **STM32CubeMX** (integrated with CubeIDE)
- HAL Library (included with STM32CubeMX)

## Configuration Steps

### 1. Create New Project

```
File → New → STM32 Project
```

1. Select your target MCU or board
2. Enter project name
3. Click Finish

### 2. Configure ADC Peripheral

Open the `.ioc` file and configure as follows:

#### Enable ADC Channel
1. Navigate to **Analog → ADC1**
2. Enable an IN channel (e.g., IN0)
3. The corresponding pin will be highlighted in the pinout view

#### ADC Parameters
Configure under **ADC1 → Parameter Settings**:

| Parameter | Value |
|-----------|-------|
| Clock Prescaler | PCLK2 divided by 4 |
| Resolution | 12 bits |
| Data Alignment | Right aligned |
| Scan Conversion Mode | Disabled |
| Continuous Conversion Mode | Enabled |
| Discontinuous Conversion Mode | Disabled |
| DMA Continuous Requests | Enabled |

#### ADC Channel Rank
Under **Rank** settings:
- **Channel**: Select your enabled channel
- **Sampling Time**: 84 Cycles (adjust based on MCU)

### 3. Configure DMA

Navigate to **ADC1 → DMA Settings** and click **Add**:

| Parameter | Value |
|-----------|-------|
| DMA Request | ADC1 |
| Direction | Peripheral to Memory |
| Priority | Medium |
| Mode | Circular |
| Increment Address | Memory |
| Data Width | Half Word (16-bit) |

### 4. Enable Interrupts (Optional)

Under **ADC1 → NVIC Settings**:
- ☑ ADC1 global interrupt (for conversion complete callback)

### 5. Generate Code

```
Project → Generate Code (Alt+K)
```

## Implementation

### Basic Setup

Add the following code to `main.c`:

```c
/* USER CODE BEGIN PV */
uint16_t adc_value = 0;  // Variable to store ADC reading
/* USER CODE END PV */

int main(void)
{
  /* MCU Configuration */
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_DMA_Init();
  MX_ADC1_Init();
  
  /* USER CODE BEGIN 2 */
  // Start ADC conversion with DMA
  HAL_ADC_Start_DMA(&hadc1, (uint32_t*)&adc_value, 1);
  /* USER CODE END 2 */

  while (1)
  {
    /* USER CODE BEGIN 3 */
    // adc_value is automatically updated by DMA
    // Process or display the value as needed
    HAL_Delay(100);
    /* USER CODE END 3 */
  }
}
```

### Voltage Conversion

Convert raw ADC value to voltage:

```c
// For 3.3V reference and 12-bit resolution
float voltage = (adc_value * 3.3f) / 4095.0f;

// For 3.0V reference and 12-bit resolution
float voltage = (adc_value * 3.0f) / 4095.0f;
```

### Conversion Complete Callback (Optional)

If you enabled ADC interrupts, implement the callback:

```c
/* USER CODE BEGIN 0 */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
  if(hadc->Instance == ADC1)
  {
    // Conversion complete
    // adc_value has been updated
    // Add your processing logic here
  }
}
/* USER CODE END 0 */
```

## Usage Examples

### Example 1: Reading a Potentiometer

```c
while (1)
{
  // Convert to percentage (0-100%)
  float percentage = (adc_value / 4095.0f) * 100.0f;
  
  // Print via UART (if configured)
  printf("Position: %.1f%%\r\n", percentage);
  
  HAL_Delay(500);
}
```

### Example 2: Temperature Sensor (LM35)

```c
while (1)
{
  // Convert to voltage
  float voltage = (adc_value * 3.3f) / 4095.0f;
  
  // LM35 outputs 10mV per °C
  float temperature_C = voltage * 100.0f;
  
  // Print temperature
  printf("Temperature: %.1f°C\r\n", temperature_C);
  
  HAL_Delay(1000);
}
```

### Example 3: Light Sensor (LDR with voltage divider)

```c
while (1)
{
  // Read raw value
  uint16_t light_level = adc_value;
  
  // Determine light condition
  if(light_level > 3000)
    printf("Bright\r\n");
  else if(light_level > 1500)
    printf("Medium\r\n");
  else
    printf("Dark\r\n");
    
  HAL_Delay(500);
}
```

## Troubleshooting

### Issue: ADC readings stuck at 0 or 4095

**Solutions:**
- Verify input voltage is within 0 to VREF+ range
- Check pin configuration in CubeMX
- Ensure proper ground connection
- Verify ADC and DMA initialization order (DMA before ADC)

### Issue: Unstable or noisy readings

**Solutions:**
- Increase sampling time in ADC configuration
- Add hardware filtering (capacitor on input)
- Average multiple readings in software:
  ```c
  #define SAMPLE_COUNT 10
  uint32_t sum = 0;
  for(int i = 0; i < SAMPLE_COUNT; i++)
  {
    sum += adc_value;
    HAL_Delay(10);
  }
  uint16_t avg_value = sum / SAMPLE_COUNT;
  ```

### Issue: DMA not updating variable

**Solutions:**
- Verify DMA is initialized before ADC (`MX_DMA_Init()` before `MX_ADC1_Init()`)
- Check DMA mode is set to Circular
- Ensure DMA Continuous Requests is enabled
- Verify `HAL_ADC_Start_DMA()` is called

### Issue: Hard fault or crash

**Solutions:**
- Verify variable is declared as `uint16_t` not `uint8_t`
- Ensure proper address alignment
- Check DMA data width matches ADC resolution setting

## Important Notes

1. **DMA Mode**: Circular mode continuously updates the same memory location, perfect for single-channel monitoring
2. **CPU Efficiency**: DMA transfers happen independently, freeing CPU for other tasks
3. **Sampling Rate**: Affected by ADC clock, sampling time, and resolution settings
4. **Multiple Channels**: For multiple channels, enable Scan Mode and configure multiple ranks
5. **Reference Voltage**: Verify VREF+ on your board (usually 3.3V, but can be 3.0V on some boards)
6. **Input Protection**: Never exceed VREF+ or apply negative voltages to ADC inputs

## Performance Characteristics

| Parameter | Typical Value |
|-----------|---------------|
| Conversion Time | 12-bit: ~1-3 μs |
| Maximum Sampling Rate | Up to 2.4 MSPS (varies by MCU) |
| DMA Transfer Time | ~2-3 clock cycles |
| CPU Overhead | Near zero (DMA handles transfers) |

## Additional Resources

- [STM32 ADC Application Note (AN3116)](https://www.st.com/resource/en/application_note/cd00258017.pdf)
- [STM32CubeIDE User Guide](https://www.st.com/resource/en/user_manual/um2609-stm32cubeide-user-guide-stmicroelectronics.pdf)
- [HAL Driver Documentation](https://www.st.com/resource/en/user_manual/dm00105879.pdf)
- [STM32 DMA Application Note (AN4031)](https://www.st.com/resource/en/application_note/dm00046011.pdf)

## License

This documentation is provided as-is for educational and development purposes.

## Contributing

Contributions, issues, and feature requests are welcome! Feel free to check the issues page.

---

**Author**: Mubasher K P  
**Last Updated**: February 2026  
**MCU Tested**: STM32F4, STM32F1 series
