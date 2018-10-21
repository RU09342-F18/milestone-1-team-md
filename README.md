# Milestone 1: Stranger Things Light Wall
For the first milestone, there were multiple goals.  The primary objective was the ability to store and transmit a message through the UART communication pins.  In this case, the messages stored and transmitted were numerical values that controlled the color and brightness of an RGB LED.  Since each board both stores and transmitts data, multiple boards were able to hook together and create a string of RGB LEDs connected to different boards, outputting individual colors, only connected by the UART pins.

## Thought Process/Explanation
Due to lack of understanding of UART, the first issue tackled was how the colors and brightness of each RGB LED was controlled.  Since the previous lab dealt with the use of PWMs, the use of PWMs seemed functional in controlling each LED in the RGB LED's brightness.  This was set up by using the total clock cycle of SMCLK as the overload or total period of the PWM and each of the capture/compare registers to control each  duty cycle or LED's brightness.  Once all of the math worked out, we were able to insert a value from 0-255 for each LED and output an expected color.  Since the PWM period was very slow (used the entire clock cycle as the period), you could see the RBG LED blink.  To fix this, the original 1 MHz clock was increased to 8 MHz. Once the hardware portion was completed, the UART communication portion was tackled.  Once the proper UART Interrupt Vector name, and RX and TX variable names were discovered, the instruction from the lab worked themselves out.  The first bit carried would be the size of the message with the following three bits being the red, green and blue values for the first LED.  Depending on the size of the message, the next bit in the series would be the end bit (0x0D) or it would another three bits signifying the intensity of the three nodes in the RGB LED.  These values would be passed in either through USB (For the first node in the series) or by the UART Transmit Pin(from the first node) to the UART Recieve Pin(the second node).  Which ever value was recieved was display in that node's RGB LED. 

# Original ReadME (delete after Appnote is finished)
For the first milestone, you will be building "Addressable" RGB LEDs which can be connected in series with one another and can have patterns generated from them. You will need to utilize any development board of your choosing to generate a RGB node. By the week of _**OCTOBER 8**_, you will be expected to come into lab with a fully operational RGB node ready to be connected together. Your node will be tested individually during that lab period, with your documentation (code and readme) being graded throughout the week.

Your grade will be broken up as follows:
* RGB Node Performance (Including Code Review) 60%
* Code Documentation 20%
* Readme 20%

## Background Theory
In the following scene from Netflix's "Stranger Things", Will Byers stuck in a parallel dimension can only communicate to his mother by causing Christmas Lights to light up. The mother then strings up a set of lights like a Ouija Board so they can communicate back and forth.

[![Stranger Things Light Wall](https://i.gyazo.com/0ec40d564af88528784912d678e72122.jpg)](https://www.youtube.com/watch?v=BlVN5Ukp7c8 "Stranger Things - Lights Wall Scene [Leg PT-BR]")

## Our Implementation

For starters, the entire network of devices will look similar to the image below.

![Network Layout](https://i.gyazo.com/9786b254eafdff55ef3b9259d1c7dbf9.png)

You will be responsible for picking a development boards as well as designing the proper driving circuitry for the RGB LED. There are no specified pins that need to be utilized, so it is up to your group to determine the proper pins for your application.

#### RGB on the G2
Your new boards have an RGB on board. As much as it would make life easy, part of the point of this Milestone is to look at interfacing these things to your boards. For this reason, you need to hook up your LED externally. If you find that the pins being used for the on-board RGB are really the only pins you can use, you can use jumpers to breakout to a breadboard.

### RGB Node

Each of these RGB Nodes will be responsible for taking in a string of Hex values over the UART RX Line, using the 3 least significant bytes as their own RGB values. These RGB values then need to be converted and transformed into Duty Cycles to replicate the proper color. Since the incoming string may be at most 80 bytes, the remaining string after removing the 3 least significant bytes needs to be transmitted over the UART TX line to the next node.

![Node Layout](https://i.gyazo.com/b9eb35f557c10cead61346034936084b.png)

The RGB Node needs to be configured to initialize all timers and peripherals upon starting up and enter a Low Power Mode while waiting for a message over UART. When a message begins to be received over UART, the first 3 bytes should be stored into an RGB buffer which can then be used to set the duty cycles for the RGB LED. The rest of the message should be placed into a UART TX buffer and when the incoming message is complete, should then be sent out over the UART TX Line. Once the message has been transmitted, you should then set your RGB LED accordingly. The flow of this code can be seen below.

![Program Flow](https://i.gyazo.com/2cd40704327b558c01df0fb9c098e0e1.png)

## UART Protocol
To make sure that everyone can talk with one another, we need to ensure that everyones UART settings are the same. For this project, your UART should be set to:
* 9600 BAUD

### Messaging Protocol
For starters, we need to establish how each of these node know how many bytes are to be expected for each message. Since each node will be using 3 bytes to set their RGB, we would need at least 3bytes\* 26 nodes = 78 bytes. For debugging purposes, we would like to have at least one byte at the end of the message so that our head node can determine whether or not each node removed the first 3 bytes from their received message. This means we will need an additional byte at the end of the message, bringing our total to 79 bytes. It would also be extremely convenient if each node could know how many bytes to expect in a message. For this we can easily add a byte to the beginning of our message to tell each node how many bytes to expect, bringing the total to 80 bytes.

| Byte Number |  Contents | Example |
| ----------- | --------- | ------- |
| Byte 0      | Number of bytes (N) including this byte | 0x50 (80 bytes) |
| Bytes 1-(N-2) | RGB colors for each node | 0xFF (red) 0x00 (green) 0x88 (blue) ... |
| Byte N-1 | End of Message Character | 0x0D (carriage return) |

#### Example communication between 2 nodes
The following would be received by your node.

| Byte Number | Content | Meaning |
| ----------- | ------- | ------- |
| Byte 0      | 0x08    | 8 total bytes in the package |
| Byte 1      | 0x7E    | Red (current node): 50% duty cycle |
| Byte 2      | 0x00    | Green (current node): 0% duty cycle |
| Byte 3      | 0xFF    | Blue (current node): 100% duty cycle |
| Byte 4      | 0x40    | Red (next node): 25% duty cycle |
| Byte 5      | 0xFF    | Green (next node): 100% duty cycle |
| Byte 6      | 0x00    | Blue (next node): 0% duty cycle |
| Byte 7      | 0x0D    | End of Message Check byte |

After taking Bytes 1-3 for use in the current node, the message would be repackaged into a new message and transmitted to the next node.

| Byte Number | Content | Meaning |
| ----------- | ------- | ------- |
| Byte 0      | 0x05    | 5 total bytes in the package |
| Byte 1      | 0x40    | Red (current node): 25% duty cycle |
| Byte 2      | 0xFF    | Green (current node): 100% duty cycle |
| Byte 3      | 0x00    | Blue (current node): 0% duty cycle |
| Byte 4      | 0x0D    | End of Message Check byte |
