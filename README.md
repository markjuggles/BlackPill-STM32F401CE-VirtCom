# Blackpill STM32F401CE CDC USB Example

*December 2020*

This example will create a USB Communications Device Class (CDC) application which is also known as a virtual com port.  It will look like a serial port to a computer.

It uses the ST toolchain which consists of two applications:

* STM32CubeMX  (Version 6.1.0)
* STM32CubeIDE (Version 1.5.0)

Both can be downloaded from ST: https://www.st.com/content/st_com/en/stm32cube-ecosystem.html

STM32CubeMX is a configuration tool for setting up a software project with the right pin, peripheral, and software libary configuration. It supports multiple IDE options including STM32CubeIDE.

STM32CubeIDE is an Integrated Development Environment which allows a programmer to edit, compile, and debug from the same tool.  It is based on the very popular Eclipse IDE and bundles gcc for ARM.

# STM32CubeMX

The first step in creating a new project is to run STM32CubeMX and configure the I/O and peripherals.

## New Project

This is the default starting place and it gives you three options:

- Create a project based on an MCU.
- Create a project based on an ST Development Board.
- Create a proejct based on an example.

For the Blackpill, you will need to choose the first option, MCU.

### MCU Selector

Type in your MCU.  Mine was the STM32F401CE.

Notice that there is some good information on this page!  

- Features
- Block Diagram
- Docs & Resources
- Datasheet

When you're done with the research, click the **[ Start Project ]** button.

## Pinout & Configuration

Note the *Categories* window on the left.  We will work our way down the categories configuring what is necessary for a USB CDC device.


## System Core

### RCC

> Set the **High Speed Clock (HSE)** to *Crystal/Ceramic Resonator*.


### Connectivity

> Set the **USB_OTG_FS** to *Device_Only*.

### Middleware

> Set the **USB_DEVICE** *Class For FS IP* to *Communication Device Class (Virtual Port Com)*.


That is all that is necessary under Pinout & Configuration.

## Clock Configuration

It is necessary for USB to have a stable clock which is why the HSE was selected in the RCC configuration. USB runs at 12 MHz but the USB peripheral requires 48 MHz.  It's not clear to me which one or ones have to be 48 MHz so in this example we set as many as possible to 48 MHz and it works.  

Note that these settings directly influence the code in SystemClock_Config().

### Input Frequency
> 25

### PLL Source Mux

> HSE


### PLL Source Mux Divider (/M)

> 25

### Main PLL Multiplier (*N)

> x 192

### Main PLL Divider (/P)

> / 4

### Main PLL Divider (/Q)

> / 4

### System Clock Mux

> PLLCLK

### APB1 Prescaler 

> / 2

This results in all clocks being 48 MHz except the APB1 peripheral which is 42 MHz max so it is set to 24 MHz.

## Project Manager

This is where you name the project and select your Toolchain/IDE.

The default Toolchain/IDE is **not** STM32CubeIDE so be sure to change it if that is what you plan to use.

## Last STM32CubeMX Step

> Click the *[ GENERATE CODE ]* button to generate the project.

Be sure to save your project so that you can update it later if you like.  Note that in the recent projects listing, the European date format is used:  DD/MM/YYYY.


# STM32CubeIDE

This is an Eclipse based IDE which has been configured to work with the STM32 family.  If you are not familiar with Eclipse then learn the basics.  It is used for NXP, TI, Silicon Labs, and others.  It is also very applicable for systems programming in C++ and Java.

Now open the project which was just created with the IDE.  You are primarily interested in Core/Src/main.c which contains the main() function.

Notice that the code is filled with several USER CODE comment pairs like these:

`/* USER CODE BEGIN <section> */`

`/* USER CODE END <section> */`

**NOTE:** It is important to keep your code betwen them so that if you modify a setting in STM32CubeMX, it will not wipe out your changes.


# STM32CubeIDE -- Transmit Data Example

You will need to add an include for string.h:
```
    /* USER CODE BEGIN Includes */
    #include <string.h>
    /* USER CODE END Includes */
```

Declare an external function from the USB library:
```
    /* USER CODE BEGIN PFP */
    extern uint8_t CDC_Transmit_FS(uint8_t* Buf, uint16_t Len);
    /* USER CODE END PFP */
```

Declare some variables at the top of main():

```
    /* USER CODE BEGIN 1 */
    uint32_t comcnt;
    char combuf[128];
    /* USER CODE END 1 */
```

Finally add the code which writes to the USB 'in' buffer:

```
  /* Infinite loop */
  /* USER CODE BEGIN WHILE */

  comcnt = 0;

  while (1)
  {
	  HAL_Delay(1000);
	  sprintf(combuf, "%8lu: Hello STM32!\n", ++comcnt);
	  CDC_Transmit_FS((uint8_t *) combuf, strlen(combuf));

    /* USER CODE END WHILE */
    
    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
```

This should compile and run with no issues.

Letting the device run for a few days results in this:

```                                                          
  169342: Hello STM32!                                                          
  169343: Hello STM32!                                                          
  169344: Hello STM32!                                                          
  169345: Hello STM32!                                                          
  169346: Hello STM32!                                                          
  169347: Hello STM32!                                                          
```

# STM32CubeIDE -- Echo Recieved Data Example

This function will be called when a packet of incoming data arrives at the device:

```
int8_t CDC_Receive_FS(uint8_t* Buf, uint32_t *Len)
```

You may (1) process the data *quickly* in this function, or (2) make a copy, or (3) swap the receive buffer.
The comments warn that if more data comes in before this function exits, it will be NAK'd so option 3 is best for high-speed data.

Here is an example with option 2, copying to a global buffer in main():

```
static int8_t CDC_Receive_FS(uint8_t* Buf, uint32_t *Len)
{
  /* USER CODE BEGIN 6 */
	extern char echoBuf[256];
	extern int  echoLen;

	memcpy(echoBuf, Buf, *Len);         /**** EXAMPLE CODE -- No test for overflow ****/
	echoLen = *Len;

  USBD_CDC_SetRxBuffer(&hUsbDeviceFS, &Buf[0]);
  USBD_CDC_ReceivePacket(&hUsbDeviceFS);
  return (USBD_OK);
  /* USER CODE END 6 */
}
```

Inside the while(1) loop in main():

```
	  if(echoLen > 0)
	  {
		  CDC_Transmit_FS((uint8_t *) echoBuf, echoLen);
		  echoLen = 0;
	  }
```

As a result, data typed into the terminal will be echoed.  If you like, you can experiment by forcing the ASCII to upper case or something.


# Programming

My evaluation uses the ST-Link-V2 which allows programming and debugging.  

If your ST-Link-V2 is still in the mail, a Segger J-Link should work.   As a last resort, you can try the on-chip bootloader.

# Default Buffers

These are the framework defined buffers which should be fine for generic applications.

```
/* Create buffer for reception and transmission           */
/* It's up to user to redefine and/or remove those define */
/** Received data over USB are stored in this buffer      */
uint8_t UserRxBufferFS[APP_RX_DATA_SIZE];

/** Data to send over USB CDC are stored in this buffer   */
uint8_t UserTxBufferFS[APP_TX_DATA_SIZE];
```

# Windows Terminal Emulators

Use whatever you like.  

- PuTTY
- Realterm
- Teraterm
- Termie
- Termite
- etc.

Some of them get very angry when you reprogram the device though.  Stopping and restarting might be necessary.  Termie is forgiving though.

**TIP:** If you are running Windows, do this before starting or pluging in your device:

```
C:\Users\STM> mode | find "COM"
Status for device COM3:
Status for device COM10:
```

**Now:** Start or plugin your device and repeat:

```
C:\Users\STM> mode | find "COM"
Status for device COM3:
Status for device COM10:
Status for device COM8:
```
COM8 was added to the list so that is my COM port this time.  This only shows available COM ports.  If the port has been opened, it will not be on the list.

Also, the baudrate does not matter.  The USB CDC is only emulating RS-232 serial.  1200 baud to 115,200 work equally.


# Reference

UM1734 - User manual - STM32Cube™ USB device library

https://www.st.com/resource/en/user_manual/dm00108129-stm32cube-usb-device-library-stmicroelectronics.pdf  


