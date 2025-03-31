# Embeddable Web Serial ESP32 Flasher (C3/C6/S2/S3)

A JavaScript library using the Web Serial API to flash ESP32-C3, ESP32-C6, ESP32-S2, and ESP32-S3 devices directly from a web browser. It features a simple interface designed for easy embedding into your HTML projects.

This flasher communicates with the ESP32's ROM bootloader and can upload a flashing stub for faster and more reliable flashing operations.

## Key Features

*   **Browser-Based Flashing:** Flash supported ESP32 chips directly from Chrome, Edge, or Opera using the Web Serial API.
*   **No External Tools:** Does not require users to install Python, esptool, or drivers (uses the browser's built-in API).
*   **Simple Interface:** Provides a straightforward JavaScript class (`ESPFlasher`) for easy integration.
*   **Embeddable:** Designed to be easily included and used within web pages and applications.
*   **Supported Chips:** Currently supports ESP32-C3, ESP32-C6, ESP32-S2, and ESP32-S3.
*   **Automatic Chip Detection:** Identifies the connected chip based on its magic value read from memory.
*   **Stub Loader:** Includes and utilizes pre-compiled flashing stubs for supported chips to enhance speed and reliability.
*   **Core Bootloader Commands:** Implements essential commands like Sync, Read/Write Register, Memory Download, and Flash operations.
*   **SLIP Protocol:** Handles SLIP encoding and decoding for serial communication framing.
*   **Utility Functions:** Includes methods for reading the MAC address, basic reliability testing, and performing a blank check.

## Supported Chips

*   ESP32-C3
*   ESP32-C6
*   ESP32-S2
*   ESP32-S3

*(Support for other chips like the original ESP32 or ESP32-S2 require adding their specific stubs and magic values).*

## Requirements

1.  **Browser:** A modern web browser that supports the Web Serial API (e.g., Google Chrome, Microsoft Edge, Opera).
2.  **Hardware:** An ESP32-C3, C6, S2, or S3 device connected via USB.
3.  **Bootloader Mode:** You may have to put the ESP32 manually into **Serial Bootloader Mode**. This is typically done by:
    *   Holding down the `BOOT` (or `IO0`) button.
    *   Pressing and releasing the `RESET` (or `EN`) button.
    *   Releasing the `BOOT` button.
    *   Alternatively, hold `BOOT` while plugging in the USB cable.

## How it Works

1.  The user initiates a connection via a button click on your web page.
2.  The browser prompts the user to select a serial port (`navigator.serial.requestPort()`).
3.  The `ESPFlasher` library opens the selected port.
4.  It attempts to synchronize with the ESP32's ROM bootloader (`SYNC` command).
5.  It reads a magic value from a specific memory address to identify the chip type.
6.  It downloads and executes a small "stub" program to the ESP32's RAM (`MEM_BEGIN`, `MEM_DATA`, `MEM_END`). This stub handles flash operations more efficiently than the ROM bootloader alone.
7.  The library then uses commands like `FLASH_BEGIN`, `FLASH_DATA`, and `FLASH_END` (communicating with the stub) to write the firmware binary to the device's flash memory.
8.  All communication is framed using the SLIP protocol.

## Usage Example

```html
<!DOCTYPE html>
<html>
<head>
    <title>ESP32 Web Flasher</title>
</head>
<body>
    <h1>ESP32 Web Flasher (C3/C6/S2/S3)</h1>

    <button id="connectButton">Connect</button>
    <input type="file" id="firmwareFile" accept=".bin">
    <button id="flashButton" disabled>Flash Firmware</button>
    <button id="disconnectButton" disabled>Disconnect</button>

    <pre id="logOutput"></pre>
    <progress id="progressBar" value="0" max="100" style="width: 100%;"></progress>

    <!-- Include the flasher script -->
    <script src="esp-flasher.js"></script>

    <script>
        const connectButton = document.getElementById('connectButton');
        const firmwareFile = document.getElementById('firmwareFile');
        const flashButton = document.getElementById('flashButton');
        const disconnectButton = document.getElementById('disconnectButton');
        const logOutput = document.getElementById('logOutput');
        const progressBar = document.getElementById('progressBar');

        const flasher = new ESPFlasher();
        let firmwareData = null;

        // --- Logging ---
        function log(message) {
            console.log(message);
            logOutput.textContent += message + '\n';
            logOutput.scrollTop = logOutput.scrollHeight; // Auto-scroll
        }
        flasher.logDebug = log; // Use browser console for debug logs
        flasher.logError = (message) => log(`ERROR: ${message}`); // Prefix errors

        // --- Event Listeners ---
        connectButton.addEventListener('click', async () => {
            log('Requesting port...');
            try {
                await flasher.openPort();
                log('Port opened. Trying to sync...');
                await flasher.sync();
                log(`Synced. Detected Chip: ${flasher.current_chip}`);

                const mac = await flasher.readMac();
                log(`MAC Address: ${mac}`);

                log('Downloading stub...');
                const stubLoaded = await flasher.downloadStub();
                if (!stubLoaded) {
                     log('Failed to download stub. Is the device locked or unsupported?');
                     await flasher.disconnect();
                     return;
                }
                log('Stub downloaded successfully.');

                connectButton.disabled = true;
                flashButton.disabled = false;
                disconnectButton.disabled = false;
                firmwareFile.disabled = false;

            } catch (error) {
                flasher.logError(`Connection failed: ${error.message || error}`);
                await flasher.disconnect(); // Ensure cleanup on error
            }
        });

        firmwareFile.addEventListener('change', (event) => {
            const file = event.target.files[0];
            if (!file) {
                firmwareData = null;
                log('No file selected.');
                return;
            }
            const reader = new FileReader();
            reader.onload = (e) => {
                firmwareData = new Uint8Array(e.target.result);
                log(`Firmware loaded: ${file.name} (${firmwareData.length} bytes)`);
            };
            reader.onerror = (e) => {
                 flasher.logError(`Failed to read file: ${e}`);
                 firmwareData = null;
            }
            reader.readAsArrayBuffer(file);
        });

        flashButton.addEventListener('click', async () => {
            if (!firmwareData) {
                flasher.logError('No firmware file selected or loaded.');
                return;
            }
            if (!flasher.port || !flasher.stubLoaded) {
                flasher.logError('Not connected or stub not loaded.');
                return;
            }

            flashButton.disabled = true; // Prevent multiple clicks
            disconnectButton.disabled = true;

            try {
                log('Starting flash process...');
                // Example: Flash firmware to address 0x10000
                // Adjust address as needed for your firmware layout
                const flashAddress = 0x10000;

                // --- Optional: Erase before flashing (example) ---
                // log('Erasing flash region (this might take a while)...');
                // NOTE: Erase command implementation is not fully shown in the provided source
                // await flasher.eraseRegion(flashAddress, firmwareData.length);
                // log('Erase complete.');

                log(`Writing ${firmwareData.length} bytes to address 0x${flashAddress.toString(16)}...`);
                progressBar.value = 0;

                await flasher.writeFlash(flashAddress, firmwareData, (bytesWritten, totalBytes) => {
                    const progress = Math.round((bytesWritten / totalBytes) * 100);
                    progressBar.value = progress;
                    // Optional: Update UI more frequently if needed
                    // log(`Progress: ${progress}%`);
                });

                progressBar.value = 100;
                log('Flash complete!');
                log('You may need to manually reset the device to run the new firmware.');

            } catch (error) {
                flasher.logError(`Flashing failed: ${error.message || error}`);
            } finally {
                 // Re-enable buttons even on failure, except connect
                 disconnectButton.disabled = false;
                 flashButton.disabled = (firmwareData == null); // Keep flash disabled if no file
            }
        });

        disconnectButton.addEventListener('click', async () => {
            await flasher.disconnect();
        });

        // Handle disconnects initiated by the library or browser
        flasher.disconnected = () => {
            log('Port disconnected.');
            connectButton.disabled = false;
            flashButton.disabled = true;
            disconnectButton.disabled = true;
            firmwareFile.disabled = true;
            firmwareData = null;
            progressBar.value = 0;
        };

    </script>
</body>
</html>
```

## API Overview (`ESPFlasher` class)

*   `constructor()`: Creates a new flasher instance.
*   `async openPort()`: Prompts the user to select a serial port and opens it.
*   `async disconnect()`: Closes the serial port and cleans up.
*   `async sync()`: Synchronizes with the ESP32 ROM bootloader and detects the chip type.
*   `async downloadStub()`: Downloads the appropriate flashing stub to the ESP32's RAM. Returns `true` on success, `false` on failure.
*   `async writeFlash(address, data, progressCallback)`: Writes the provided `Uint8Array` data to the specified flash address. `progressCallback(bytesWritten, totalBytes)` is called periodically.
*   `async readMac()`: Reads and returns the MAC address of the connected device as a string (e.g., "AA:BB:CC:11:22:33").
*   `async readReg(address)`: Reads a 32-bit value from the specified memory/register address. Returns the value as a number.
*   `async testReliability(callback)`: Performs repeated register reads for a short duration to test connection stability. `callback(progressPercentage)` is called periodically. Returns `true` if successful, `false` on error or mismatch.
*   `async blankCheck(callback)`: Reads through the flash memory to check how much of it is erased (contains `0xFF`). `callback(currentAddress, startAddress, endAddress, blockSize, erasedBytesInBlock, totalErasedBytes)` is called after each block read.
*   `logDebug`: Assign a function `(message) => { ... }` to handle debug logging. Defaults to no-op.
*   `logError`: Assign a function `(message) => { ... }` to handle error logging. Defaults to no-op.
*   `devMode`: Set to `true` to enable verbose packet logging to the browser console.
*   `disconnected`: Assign a function `() => { ... }` to be called when the port is disconnected unexpectedly or via `disconnect()`.
*   `current_chip`: (Read-only property) String identifier of the detected chip ("esp32c3", "esp32s3", etc.) after a successful `sync()`.
*   `stubLoaded`: (Read-only property) Boolean indicating if the flashing stub has been successfully loaded after `downloadStub()`.

*(Note: Some less common bootloader commands like `ERASE_FLASH`, `ERASE_REGION`, `SPI_ATTACH`, `CHANGE_BAUDRATE`, etc., are defined as constants but might not have dedicated high-level methods implemented in this version.)*

## Limitations

*   **Chip Support:** Only the listed chips (C3, C6, S2, S3) are currently supported due to the included stubs and detection logic.
*   **Browser Support:** Requires a browser with Web Serial API support.
*   **Error Handling:** While basic error handling exists, it might be less robust than dedicated tools like `esptool.py`.
*   **Advanced Features:** Does not currently support features like flash encryption, secure boot interactions, or eFuse programming.
*   **Stub Size:** Embedding stub data directly in the JS increases the initial script size.

## License

```
MIT License

Copyright (c) 2025 g3gg0

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```


