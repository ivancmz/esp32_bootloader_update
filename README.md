# ESP32-S2 Bootloader update for fixing Flash SPI frequency

As described in the [related issue](https://github.com/espressif/esp-idf/issues/15849), ESP32-S2 devices flashed with bootloader configured for 40MHz SPI flash frequency might experience SSL connection failures due to cryptographic operations timing out. Devices flashed with 80MHz SPI flash frequency work correctly. This is true at least for ESP-IDF v4.4, I'm working on reproducing the issue on the latest IDF version, but our device fleet is still on v4.4.

This presents a challenge for devices already deployed in the field that were initially programmed with the 40MHz configuration, especially when physical access for reflashing is limited or impossible.

## Workarround: OTA Bootloader Fix

To answer my own question, I have developed a workaround that allows updating the bootloader via OTA to change the flash frequency from 40MHz to 80MHz without requiring physical access to the device.

### Implementation

The solution embeds a correctly configured bootloader binary within the application firmware and deploys a routine that checks and updates the bootloader if necessary.

#### 1. Prepare the Bootloader Binary

1. Compile the bootloader with 80MHz flash frequency configured
2. Copy the resulting `bootloader.bin` to your `main` folder

#### 2. Embed the Bootloader in Firmware

Add the following to your `main/CMakeLists.txt` to embed the bootloader binary into your firmware:

```cmake
idf_component_register(
    SRCS "main.c" "other_files.c"
    INCLUDE_DIRS "."
    # Other options...
    EMBED_FILES bootloader.bin
)
```

#### 3. Enable Dangerous Flash Writes

In `menuconfig`, enable the option to write to the bootloader area:

```
Component config > SPI Flash driver > Allow dangerous write ops: Allowed (CONFIG_SPI_FLASH_DANGEROUS_WRITE_ALLOWED)
```

#### 4. Bootloader Fix Implementation

Add the following code to your application:

```c
#include "esp_log.h"
#include "esp_system.h"
#include "esp_spi_flash.h"

// Bootloader header signatures for different flash frequencies
#define FLASH_SPEED_80MHZ 0x2F0203E9  // E9,03,02,2F in memory (little-endian)
#define FLASH_SPEED_40MHZ 0x200203E9  // E9,03,02,20 in memory (little-endian)

// External references to the embedded bootloader binary
extern const uint8_t bootloader_bin_start[] asm("_binary_bootloader_bin_start");
extern const uint8_t bootloader_bin_end[]   asm("_binary_bootloader_bin_end");

void bootloader_fix(void)
{
    uint32_t header = 0;
    esp_err_t err = spi_flash_read(0x1000, &header, 4);
    
    if(err == ESP_OK) {
        ESP_LOGI("BOOT", "bootloader_fix: %08X", header);
        
        // Check bootloader header for flash frequency configuration
        if(header == FLASH_SPEED_80MHZ)
            ESP_LOGI("BOOT", "Flash speed: 80 MHz");
        else if(header == FLASH_SPEED_40MHZ)
            ESP_LOGI("BOOT", "Flash speed: 40 MHz");
        else
            ESP_LOGI("BOOT", "bootloader_fix: bootloader unknown %08X", header);
        
        // Update bootloader if not already at 80MHz
        if(header != FLASH_SPEED_80MHZ) {
            do {
                // Erase bootloader section
                err = spi_flash_erase_range(0x1000, 0x8000-0x1000);
                if (err != ESP_OK) {
                    ESP_LOGE("BOOT", "bootloader_fix: Failed to erase flash sector 0x1000");
                    continue;
                } else {
                    ESP_LOGI("BOOT", "bootloader_fix: Flash sector 0x1000 erased");
                }
                
                // Write new bootloader
                if (spi_flash_write(0x1000, bootloader_bin_start, 
                                   bootloader_bin_end - bootloader_bin_start) != ESP_OK) {
                    ESP_LOGE("BOOT", "bootloader_fix: Failed to write flash header");
                    continue;
                } else {
                    ESP_LOGI("BOOT", "bootloader_fix: Flash speed fixed to 80 MHz");
                    esp_restart();  // Restart to apply the new bootloader
                }
            } while(1);  // Retry until successful
        }
    }
}
```

#### 5. Call from Application

Call the bootloader fix routine at the beginning of your application:

```c
void app_main(void) 
{
    bootloader_fix();
    
    // Rest of the application...
}
```

## Safety Considerations

1. The bootloader update process takes approximately 500ms, during which power interruption could potentially brick the device. However, testing has shown this to be reliable across multiple devices.

2. The update is only performed if needed (40MHz → 80MHz), reducing the risk for devices that are already correctly configured.

3. The retry loop ensures that the process completes successfully, even if initial attempts fail.

4. Embedding the bootloader binary directly in the firmware ensures integrity of the update.

## Limitations

1. This approach requires that the device can still connect to the OTA server to receive the updated firmware containing this fix routine.

2. For devices that have already lost connectivity due to SSL failures, physical access for reflashing is still required.

3. This solution assumes that the flash chip itself is capable of operating at 80MHz (which is true for most modern flash chips used with ESP32-S2).

## Testing and Validation

This solution has been successfully tested on multiple ESP32-S2 devices that were previously flashed with 40MHz bootloaders. All devices were able to update their bootloaders and subsequently restored full SSL/TLS functionality.

## Memory Considerations

The embedded bootloader binary adds approximately 16-24KB to the firmware size. Ensure your partitioning scheme can accommodate this increased size.

## References
- [ESP-IDF SPI Flash Access](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/spi_flash/index.html#spi-flash-access-api)
- [ESP-IDF Config SPI Dangerous Write](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/kconfig-reference.html#config-spi-flash-dangerous-write)
- [ESP-IDF Embedding binary data](https://docs.espressif.com/projects/esp-idf/en/v4.4.7/esp32/api-guides/build-system.html?highlight=build%20system#embedding-binary-data)
