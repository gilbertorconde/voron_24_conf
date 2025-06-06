# Setting Up Two Webcams on Raspberry Pi CM4 (Manta M8P) for Klipper

This guide provides a robust solution for running two USB webcams on a Raspberry Pi Compute Module 4 (CM4), especially when paired with a board like the BTT Manta M8P. It addresses two primary challenges: inconsistent `/dev/videoX` naming and potential USB bandwidth saturation.

The information related with the kernel quirk was originally from this investigation:
[https://github.com/mainsail-crew/crowsnest/issues/109#issuecomment-2028029817](https://github.com/mainsail-crew/crowsnest/issues/109#issuecomment-2028029817
)

## 1\. Ensuring Consistent Webcam Naming with `udev` Rules

Linux dynamically assigns `/dev/videoX` names (e.g., `/dev/video0`, `/dev/video1`) based on detection order, which can change after reboots. To fix this, we'll use `udev` rules to create consistent, custom symlinks (e.g., `/dev/webcam1`, `/dev/webcam2`).

### 1.1. Identify Unique USB Paths for Each Webcam

This is the most critical step. You need to identify the unique physical USB path for each webcam.

1.  **Unplug all webcams** from your Raspberry Pi.
    
2.  **Plug in ONLY your first webcam in it's final destination USB port** (the one you want to be `/dev/webcam1`).
    
3.  Open a terminal on your Raspberry Pi and run the following command to get its unique attributes, specifically focusing on its `KERNELS` (USB path), `idVendor`, and `idProduct`:
    
    bash
    
    ```
    udevadm info --name=/dev/video0 --attribute-walk | grep -E 'KERNELS|idVendor|idProduct'
    ```
    
    (Note: It might show as `/dev/video0` or `/dev/video1` depending on other video devices. Just ensure only one webcam is plugged in).
    
    Look for the specific line representing the webcam's device path. It should look something like:
    
    text
    
        KERNELS=="1-1.1.6:1.0" # This is the specific device path for webcam1
        ATTRS{idProduct}=="0345"
        ATTRS{idVendor}=="0ac8"
    
    **Make a note of the `KERNELS` value and the `idVendor`/`idProduct` for this webcam.**
    
4.  **Unplug the first webcam.**
    
5.  **Plug in ONLY your second webcam in it's final destination USB port** (the one you want to be `/dev/webcam2`).
    
6.  Repeat the `udevadm info` command:
    
    bash
    
    ```
    udevadm info --name=/dev/video0 --attribute-walk | grep -E 'KERNELS|idVendor|idProduct'
    ```
    
    Again, look for its unique `KERNELS` value, `idVendor`, and `idProduct`. It should resemble:
    
    text
    
        KERNELS=="1-1.3:1.0" # This is the specific device path for webcam2
        ATTRS{idProduct}=="0345"
        ATTRS{idVendor}=="0ac8"
    
    **Make a note of these values for your second webcam.**
    
    _(If both cameras share `idVendor` and `idProduct` (2 identical cameras), which makes the `KERNELS` or `DEVPATH` crucial for differentiation)._
    

### 1.2. Create the `udev` Rules File

Now, you'll create a `udev` rule file that uses these unique identifiers to create symlinks.

1.  Open the `udev` rules file in a text editor:
    
    bash
    
    ```
    sudo nano /etc/udev/rules.d/99-webcam.rules
    ```
    
2.  Add the following rules to the file. **Ensure the `KERNELS` and `ATTRS` values exactly match what you found in the previous step for each camera.**
    
    udev
    
    ```
    # Rule for Webcam 1 (plugged into specific port on hub, path 1-1.1.6)
    # Vendor ID: 0ac8, Product ID: 0345
    SUBSYSTEM=="video4linux", KERNEL=="video*", ATTRS{idVendor}=="0ac8", ATTRS{idProduct}=="0345", DEVPATH=="*1-1.1.6/1-1.1.6:1.0*", SYMLINK+="webcam1"
    
    # Rule for Webcam 2 (plugged into specific port on hub, path 1-1.3)
    # Vendor ID: 0ac8, Product ID: 0345
    SUBSYSTEM=="video4linux", KERNEL=="video*", ATTRS{idVendor}=="0ac8", ATTRS{idProduct}=="0345", DEVPATH=="*1-1.3/1-1.3:1.0*", SYMLINK+="webcam2"
    ```
    
    *   **Explanation:**
        *   `SUBSYSTEM=="video4linux"`: Targets video devices.
        *   `KERNEL=="video*"`: Matches kernel device names starting with "video".
        *   `ATTRS{idVendor}=="0ac8"` & `ATTRS{idProduct}=="0345"`: Match your webcam's specific Vendor and Product IDs.
        *   `DEVPATH=="*1-1.1.6/1-1.1.6:1.0*"`: This is the critical part if you have identical cameras, matching the unique physical path where the webcam is connected.
        *   `SYMLINK+="webcam1"`: Creates the symbolic link `/dev/webcam1`.
3.  Save the file (Press `Ctrl+X`, then `Y` to confirm, then `Enter`).
    

### 1.3. Reload `udev` Rules and Verify

1.  Reload `udev` rules:
    
    bash
    
    ```
    sudo udevadm control --reload-rules
    sudo udevadm trigger
    sudo udevadm settle
    ```
    
2.  Plug both webcams into the _same physical USB ports_ you identified in step 1.1.
3.  Verify the symlinks are created:
    
    bash
    
    ```
    ls -l /dev/webcam*
    ```
    
    You should see output similar to:
    
    bash
    ```
    lrwxrwxrwx 1 root root 6 Jun  3 15:44 /dev/webcam1 -> video0
    lrwxrwxrwx 1 root root 6 Jun  3 15:44 /dev/webcam2 -> video2
    ```
    
    (The `-> videoX` part will still be dynamic, but `webcam1` and `webcam2` will consistently point to the correct physical camera.)

### 1.4. Udev and Crowsnest boot starting time issues

If, after reboot the `/dev/webcam*` assignments seems to be wrong (pointing to wrong `dev/video*`) it's possible that Crowsnest is starting before all webcam devices and symlinks are reliably in place. To fix this issue we need to delay Crowsnest start.

1. reate a systemd override:
```bash
sudo systemctl edit crowsnest
```

2. Add this under the `[Service]` section:
```ini
ExecStartPre=/bin/bash -c 'for i in {1..10}; do [ -e /dev/webcam1 ] && [ -e /dev/webcam2 ] && exit 0; sleep 1; done; exit 1'
```
This will wait up to 10 seconds for both symlinks to exist before starting Crowsnest.

3. Reload systemd and reboot:
```bash
sudo systemctl daemon-reload
sudo systemctl restart crowsnest
```

## 2\. Addressing USB Bandwidth Saturation (UVC Quirk)

Some cameras (ex: OV5640) might request maximum USB bandwidth regardless of the configured resolution, leading to saturation when multiple cameras are used. The Linux UVC (USB Video Class) driver has a `UVC_QUIRK_FIX_BANDWIDTH` for this.

### 2.1. Enable the `UVC_QUIRK_FIX_BANDWIDTH` Quirk

1.  The `UVC_QUIRK_FIX_BANDWIDTH` quirk has a hexadecimal value of `0x80` (decimal `128`). Optionally we can activate another quirk on top of these one to further limit the frame rate. The `UVC_QUIRK_RESTRICT_FRAME_RATE` uses the value `0x280` (decimal `640`).
2.  Create a `modprobe.d` configuration file to make this quirk persistent:
    
    bash
    
    ```
    sudo nano /etc/modprobe.d/uvcvideo.conf
    ```
    
3.  Add the following single line to the file:
    
    text
    ```
    options uvcvideo quirks=0x80
    # or optionally:
    # options uvcvideo quirks=0x280
    ```
    
4.  Save the file (Ctrl+X, Y, Enter).

### 2.2. Reload the `uvcvideo` Kernel Module

For the quirk to take effect immediately, you need to unload and then reload the `uvcvideo` kernel module.

1.  **Stop any services using the webcams** (e.g., Klipper's Moonraker, Crowsnest):
    
    bash
    
    ```
    sudo systemctl stop moonraker
    sudo systemctl stop crowsnest
    ```
    
2.  Remove the `uvcvideo` module:
    
    bash
    
    ```
    sudo rmmod uvcvideo
    ```
    
3.  Load the `uvcvideo` module with the new quirk:
    
    bash
    
    ```
    sudo modprobe uvcvideo
    ```
    

### 2.3. Configure Camera Streaming Software for YUYV Format

The `UVC_QUIRK_FIX_BANDWIDTH` often works best when the cameras are streaming in `YUYV` format, which is an uncompressed raw video format that might allow better bandwidth management by the driver.

1.  **Edit your Crowsnest configuration file** (usually `/etc/crowsnest.conf`).
    
2.  For each webcam instance, modify the configuration to use the `YUYV` format.
    
    ini
    
    ```
    [cam 1]
    # ... other settings ...
    device: /dev/webcam1
    # For ustreamer mode:
    custom_flags: --format=YUYV
    # For camera-streamer mode:
    # custom_flags: --camera-format=YUYV
    # Or, if your crowsnest version directly supports it:
    # format: YUYV
    # Consider lower resolutions/FPS (e.g., 640x480@15fps or 320x240@30fps) for two cams.
    # resolution: 640x480
    # fps: 15
    
    [cam 2]
    # ... other settings ...
    device: /dev/webcam2
    # For ustreamer mode:
    custom_flags: --format=YUYV
    # For camera-streamer mode:
    # custom_flags: --camera-format=YUYV
    # Or, if your crowsnest version directly supports it:
    # format: YUYV
    # resolution: 640x480
    # fps: 15
    ```
    
    **Important:** `YUYV` typically offers lower achievable frame rates and resolutions compared to MJPEG due to its higher raw data size. Experiment with resolutions like `640x480` or `320x240` and lower FPS (e.g., `10-15 FPS`) for stable dual-camera operation.
    
3.  **Restart your camera streaming service and Moonraker:**
    
    bash
    
    ```
    sudo systemctl restart crowsnest
    sudo systemctl restart moonraker
    ```
    

## 3\. Raspberry Pi CM4 USB Controller Configuration

On CM4-based boards like the Manta M8P, all external USB ports share a single internal USB 2.0 host controller (the XHCI controller). This means you cannot resolve bandwidth contention by plugging cameras into different USB buses, as they are all ultimately connected to the same internal controller.

Ensure your `/boot/firmware/config.txt` is correctly configured for the CM4's USB host mode.

### 3.1. Verify `/boot/firmware/config.txt`

1.  Open your `config.txt` file:
    
    bash
    
    ```
    sudo nano /boot/firmware/config.txt
    ```
    
2.  **Confirm the following lines are present and correctly configured:**
    
    *   **CM4 USB Host Mode (Crucial for Manta M8P):**  
        Look for a `[cm4]` section, and specifically the `otg_mode=1` line:
        
        text
        
        ```
        [cm4]
        # Enable host mode on the 2711 built-in XHCI USB controller.
        # This line should be removed if the legacy DWC2 controller is required
        # (e.g. for USB device mode) or if USB support is not required.
        otg_mode=1
        ```
        
        **If you have `otg_mode=1` under `[cm4]`, then your USB controller is correctly configured for modern host mode. Do NOT add `dtoverlay=dwc2,dr_mode=host`, as this would enable a conflicting legacy controller.**
3.  Save and exit if you made any changes.
    

## 4\. Final Verification

1.  **Perform a full system reboot:**
    
    bash
    
    ```
    sudo reboot
    ```
    
2.  After reboot, verify:
    *   `ls -l /dev/webcam*` still shows your consistent symlinks.
    *   `cat /sys/module/uvcvideo/parameters/quirks` outputs `128` (or `0x80`), confirming the quirk is active.
    *   Check your Klipper interface (Mainsail/Fluidd) to ensure both camera streams are present and stable.

## 5\. Troubleshooting & Debugging

If you encounter issues:

*   **Check kernel messages:**
    
    bash
    
    ```
    dmesg | grep -i "usb\|uvc\|bandwidth\|overflow"
    ```
    
    This command helps identify USB-related errors, UVC driver issues, or signs of bandwidth saturation.
*   **Isolate Cameras:** Try running one camera at a time to ensure each camera itself is functional.
*   **Experiment with Resolutions/FPS:** If two cameras are struggling, further reduce their resolution and frame rate settings in your streaming software.