| Supported Targets | ESP32 |
| ----------------- | ----- |

# I2C SSD1306 OLED Component

## I. Overview

The I2C SSD1306 OLED Component is a lightweight, efficient driver designed to interface with OLED displays using the SSD1306 controller via I2C. Although thoroughly tested on the ESP32 DEV MODULE (38 and 30-pin variants) with a 128x64, 0.96" OLED display, you may need to make minor adjustments for other ESP boards or display configurations. Exercise caution when deploying on unverified hardware setups.

### Component Background

This component is a refined evolution of an earlier version available [here](https://github.com/quackonaut/ESP-IDF-I2C_SSD1306). The primary distinction is that the former relied on the deprecated `i2c.h` header. In contrast, this updated implementation employs the modern `i2c_master.h` driver, ensuring both enhanced functionality and future compatibility with ESP-IDF releases.

**Legacy ESP-IDF I2C Driver:**

- `i2c.h`, is the header file of legacy I2C APIs.

**New ESP-IDF I2C Driver:**

- `i2c_master.h`, is the header file that provides standard communication mode specific APIs (for apps using new driver with master mode).

- `i2c_slave.h`, is the header file that provides standard communication mode specific APIs (for apps using new driver with slave mode).

## II. Integration into Your Project

Follow the steps below to seamlessly integrate this component into your ESP-IDF project.

### 1. Hardware Setup

- **OLED Connection**:  
    By default, the ESP32 uses **GPIO22** for `SCL` (clock) and **GPIO21** for `SDA` (data). Connect these pins to the corresponding SCL and SDA lines on your OLED module. If you wish to use different GPIO pins, verify that they are free and not assigned to other functions. Consult your ESP32 board’s pinout diagram for guidance.

- **Pull-Up Resistors**:  
    To maintain stable I2C communication, attach pull-up resistors to both the SCL and SDA lines. If the chosen GPIO pins lack internal pull-ups, you must add external ones (typically 4.7kΩ).

### 2. Adding the Component as a Dependency

To integrate the component into your project, you must execute de following lines in the project terminal:
```bash
idf.py add-dependency --git https://github.com/quackonauty/ESP-IDF_SSD1306.git --git-ref lastest quackonauty/ssd1306
```
Or include it as a dependency in the idf_component.yml file, which is typically located in the project's main directory. If this file does not exist, simply create a new one with that name and add the component dependency as shown below:

```yaml
# idf_component.yml
dependencies:
  quackonauty/ssd1306:
    git: https://github.com/quackonauty/ESP-IDF_SSD1306.git
    version: lastest
```

## III. Component Workflow and Usage

### 1. I2C Master Initialization

Configure the I2C master bus with the following parameters:

``` c
/* I2C Master Configuration */
static const i2c_master_bus_config_t i2c_master_bus_config = {
    .i2c_port = I2C_NUM_0,
    .scl_io_num = GPIO_NUM_22,
    .sda_io_num = GPIO_NUM_21,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .glitch_ignore_cnt = 7,
    .flags.enable_internal_pullup = true
};
static i2c_master_bus_handle_t i2c_master_bus;
```
### 2. SSD1306 Driver Initialization

Once the I2C bus is set up, initialize the SSD1306 display using the configuration below:

``` c
/* SSD1306 Configuration */
static const i2c_ssd1306_config_t i2c_ssd1306_config = {
    .i2c_device_address = 0x3C,
    .i2c_scl_speed_hz = 400000,
    .width = 128,
    .height = 64,
    .wise = SSD1306_TOP_TO_BOTTOM
};
static i2c_ssd1306_handle_t i2c_ssd1306;
```
Integrate the initialization into your application’s entry point:

``` c
void app_main(void)
{
    // Initialize I2C master bus
    ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_master_bus_config, &i2c_master_bus));
    
    // Initialize SSD1306 display
    ESP_ERROR_CHECK(i2c_ssd1306_init(i2c_master_bus, i2c_ssd1306_config, &i2c_ssd1306));
}
```

### 3. Functions and Methods of the Driver

The driver is designed to perform a wide range of tasks, from drawing individual pixels to printing text, integers, floating-point numbers, and even images. It operates by creating a structure `i2c_ssd1306_handle_t` that acts as an object, which contains an internal buffer composed of pages and segments. This buffer is dynamically sized based on the dimensions of the OLED display being used.

For more technical details, the datasheet for the SSD1306 controller can be found [here](https://cdn-shop.adafruit.com/datasheets/SSD1306.pdf).

To update the display, the workflow is divided into two main groups of functions:

**Functions for Modifying the Internal Buffer**

These functions allow you to manipulate the internal buffer, where all graphical and textual data is first written. The internal buffer represents the screen content in memory before it is displayed.

  - `i2c_ssd1306_buffer_check`: Prints the hexadecimal values of each segment in every page of the internal buffer. This is useful for debugging and verifying the current buffer state.

  - `i2c_ssd1306_buffer_clear`: Clears the entire buffer by setting all segments in every page to `0x00`.

  - `i2c_ssd1306_buffer_fill`: Fills the entire buffer with `0xFF` (all bits set), turning on all pixels.

  - `i2c_ssd1306_buffer_fill_pixel`: Sets or clears a specific pixel in the buffer at the provided `(x, y)` coordinates.

  - `i2c_ssd1306_buffer_fill_space`: Fills or clears a rectangular region of pixels within the buffer. The region is defined by the top-left coordinates `(x1, y1)` and the bottom-right coordinates `(x2, y2)`.

  - `i2c_ssd1306_buffer_text`: Renders a text string into the buffer using an 8x8 font. The text is drawn starting at the specified `(x, y)` coordinates. Optionally, the text can be inverted.

  - `i2c_ssd1306_buffer_int`: Converts an integer value to text (using an 8x8 font) and renders it into the buffer at the specified `(x, y)` coordinates. The text may be optionally inverted.

  - `i2c_ssd1306_buffer_float`: Converts a floating-point value to text with a given number of decimal places (using an 8x8 font) and writes it into the buffer starting at the specified coordinates. Optionally, the text can be rendered inverted.

  - `i2c_ssd1306_buffer_image`: Copies an image into the buffer, starting at the given `(x, y)` coordinates. The dimensions of the image are defined by its width and height. Optionally, the image can be inverted.

**Functions for Transferring the Buffer to the SSD1306 RAM**

Once the internal buffer is updated, this second group of functions is used to send the buffer data to the SSD1306 controller’s RAM. These functions are responsible for ensuring that the OLED display accurately reflects the contents of the internal buffer.

- `i2c_ssd1306_segment_to_ram`: Transfers a specific segment from a page in the internal buffer to the corresponding location in the SSD1306 display's RAM.

- `i2c_ssd1306_segments_to_ram`: Transfers a range of segments—from the initial segment to the final segment—from a page in the internal buffer to the display's RAM. This enables updating multiple contiguous segments simultaneously.

- `i2c_ssd1306_page_to_ram`: Transfers the entire buffer of a single page to the SSD1306's RAM, updating the display for that page.

- `i2c_ssd1306_pages_to_ram`: Transfers a range of pages from the internal buffer to the SSD1306's RAM, updating multiple pages in one operation.
- `i2c_ssd1306_buffer_to_ram`: Transfers the entire internal buffer (all pages) to the SSD1306's RAM, ensuring the display is fully updated with the current buffer content.

## IV. Basic Example

```c
#include <stdio.h>
#include <esp_log.h>
#include <driver/i2c_master.h>
#include <ssd1306.h>

static const uint8_t ssd1306_esp_logo_img[4][32] = {
    {0x00, 0x00, 0x00, 0xC0, 0x60, 0x18, 0x00, 0x00, 0x70, 0x78, 0x78, 0x78, 0xF8, 0xF8, 0xF0, 0xF0,
     0xF2, 0xE6, 0xE6, 0xCE, 0x9E, 0x9C, 0x3C, 0x78, 0xF8, 0xF0, 0xE0, 0xC0, 0x80, 0x00, 0x00, 0x00},
    {0x00, 0xFC, 0x07, 0x60, 0xF8, 0xFC, 0xFE, 0xFE, 0x9E, 0x9E, 0x9E, 0x3E, 0x3E, 0x7C, 0x7C, 0xF9,
     0xF9, 0xF3, 0xE7, 0xCF, 0x9F, 0x3F, 0x7F, 0xFE, 0xFC, 0xF1, 0xE3, 0x8F, 0x1F, 0xFE, 0xF8, 0x00},
    {0x00, 0x07, 0x3C, 0xE0, 0x81, 0x03, 0x07, 0xC7, 0xE7, 0xC7, 0xCF, 0x1F, 0x7F, 0xFE, 0xFC, 0xF8,
     0xE1, 0x07, 0x3F, 0xFF, 0xFF, 0xFE, 0xF0, 0x01, 0x0F, 0xFF, 0xFF, 0xFF, 0x3C, 0x00, 0x00, 0x00},
    {0x00, 0x00, 0x00, 0x00, 0x01, 0x03, 0x06, 0x0D, 0x19, 0x11, 0x30, 0x20, 0x24, 0x4F, 0x4F, 0x4F,
     0x4F, 0x40, 0x40, 0x4F, 0x4F, 0x6F, 0x27, 0x20, 0x10, 0x10, 0x08, 0x0C, 0x04, 0x00, 0x00, 0x00},
};

/* I2C Master */
static const i2c_master_bus_config_t i2c_master_bus_config = {
    .i2c_port = I2C_NUM_0,
    .scl_io_num = GPIO_NUM_22,
    .sda_io_num = GPIO_NUM_21,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .glitch_ignore_cnt = 7,
    .flags.enable_internal_pullup = true};
static i2c_master_bus_handle_t i2c_master_bus;

/* SSD1306 */
static const i2c_ssd1306_config_t i2c_ssd1306_config = {
    .i2c_device_address = 0x3C,
    .i2c_scl_speed_hz = 400000,
    .width = 128,
    .height = 64,
    .wise = SSD1306_TOP_TO_BOTTOM};
static i2c_ssd1306_handle_t i2c_ssd1306;

void app_main(void)
{
    ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_master_bus_config, &i2c_master_bus));
    ESP_ERROR_CHECK(i2c_ssd1306_init(i2c_master_bus, i2c_ssd1306_config, &i2c_ssd1306));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_image(&i2c_ssd1306, 48, 16, (const uint8_t *)ssd1306_esp_logo_img, 32, 32, false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_to_ram(&i2c_ssd1306));
    vTaskDelay(3000 / portTICK_PERIOD_MS);

    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_clear(&i2c_ssd1306));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 12, 0, "Hello, World!", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_fill_space(&i2c_ssd1306, 0, 127, 8, 8, true));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 10, "ABCDEFGHIJKLMNOP", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 18, "QRSTUVWXYZabcdfe", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT( i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 26, "ghijklmnopqrstuv", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 34, "wxyz1234567890!(", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 42, ")-=+[]{};:'\",.<>", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 50, "?/\\|_`~@#$%^&*", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_fill_space(&i2c_ssd1306, 0, 127, 58, 63, true));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_to_ram(&i2c_ssd1306));
    vTaskDelay(3000 / portTICK_PERIOD_MS);

    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_clear(&i2c_ssd1306));
    int x = 1234567890;
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_int(&i2c_ssd1306, 24, 0, x, false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_float(&i2c_ssd1306, 20, 8, x / 100000.0, 5, false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_image(&i2c_ssd1306, 48, 20, (const uint8_t *)ssd1306_esp_logo_img, 32, 32, true));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_fill_space(&i2c_ssd1306, 0, 127, 58, 63, true));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_to_ram(&i2c_ssd1306));
}
```

## V. Convert an Image to a C Array for OLED Display with Python

To convert an image into a C array that is compatible with this component, you can utilize a Python script specifically developed for this purpose. For more details, please refer to the repository available [here](https://github.com/quackonauty/ESP-IDF_PY-ImageToCArray).


## VI. Do you have any questions, suggestions, or have you found any errors?

If you have any questions or suggestions about the operation of the component, or if you encountered any errors while compiling it on your ESP32 board, please don't hesitate to leave your comment.