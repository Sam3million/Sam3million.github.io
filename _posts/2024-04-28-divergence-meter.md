---
layout: post
title: "Divergence Meter"
date: 2024-04-28
thumbnail: "/assets/divmeter/divmeter.jpg"
category: engineering
---

This post details my experience creating a replica divergence meter from the show Stein's Gate.
It uses 8 high voltage nixie tube digit displays and can display any number you wish.
Most of the time it acts as a clock, but every so often significant world events can cause
the world line to change... 

![Divergence Meter](/assets/divmeter/divmeter.jpg)
*Divergence Meter*

# Background
The Divergence Meter is a device brought back from the future which measures the current world
line's divergence from the one in which it was first built. Those with the Reading Steiner ability
can use it to determine if their actions have caused a change in the world line.

... At least that's how it goes in the show.

You might be wondering what compelled me to undertake this project. In short: Nixie tubes look
awesome. There's something about the amber glow that just can't be captured with 7 segment displays.
I had watched Stein's Gate a few years ago, and then I randomly got recommended a <a href="https://www.youtube.com/watch?v=ncqDI6mUXd4">video by Tom Titor on youtube</a>
where he built a replica Divergence Meter. I thought the project would provide good experience with PCB design,
embedded coding, and product manufacturing, so I went ahead and started planning. I followed <a href="http://www.brotoro.com/dm/index.html">Tom Titor's
website</a> as a guideline for the circuits and exterior, but designed the PCBs on my own and
coded the software completely from scratch in C.

# Parts
The choice of Nixie Tube was surprisingly complex. There are many different variations
that each have benefits and tradeoffs. The most important things to me were:
- That it had a dedicated 5 digit (some use an upside down 2)
- That it had a dedicated decimal point digit
- That it could reasonably be acquired (some are very difficult to find)

This narrowed down my choice to the IN-8-2 tube. This Soviet-era tube was manufactured
from the 1970s to the 1990s, being used in measuring equipment. New old stock can still
be found, and I was able to acquire 8 for $120 on Etsy.

<div style="display: flex; gap: 10px; align-items: flex-start;">
  <div style="flex: 1;">
    <div style="height: 500px; overflow: hidden; border-radius: 8px; background-color: transparent; -webkit-mask-image: -webkit-radial-gradient(white, black);">
      <img src="/assets/divmeter/IN-14.jpg" alt="IN-14" 
           style="width: 100%; height: 101%; object-fit: cover; object-position: center; display: block; margin: 0;">
    </div>
    <span style="display: block; text-align: center; font-size: 0.85em; margin-top: 8px;">
      IN-14 Nixie Tube With Upside-Down 2
    </span>
  </div>

  <div style="flex: 1;">
    <div style="height: 500px; overflow: hidden; border-radius: 8px; background-color: transparent; -webkit-mask-image: -webkit-radial-gradient(white, black);">
      <img src="/assets/divmeter/IN-8-2.jpg" alt="IN-8-2" 
           style="width: 100%; height: 101%; object-fit: cover; object-position: center; display: block; margin: 0;">
    </div>
    <span style="display: block; text-align: center; font-size: 0.85em; margin-top: 8px;">
      IN-8-2 Nixie Tube With Proper 5
    </span>
  </div>
</div>

## Component Breakdown
#### Electronics
- 8 IN-8-2 Nixie Tubes
- Custom Logic PCB
- Custom Nixie PCB
- PIC16F628A Microcontroller
- DS3232SN# Real Time Clock Chip
- Coin Cell Battery Connector
- Coin Cell Battery (for RTC)
- 3 HV Serial to Parallel Converters
- 1364 HVPS Horizontal (High Voltage)
- 2 Control Buttons
- On-Off Switch
- Plastic Switched Power Jack
- Connector Pins (Male & Female)
- Trimpot for HVPS
- Fuse
- Various Resistors
- Various Capacitors

#### Exterior Shell
- Perfboard
- Aluminum Sheets
- Long Hex Nuts
- Hex Screws
- Normal Screws
- Decorative Plastic Corner Covers
- Nixie Tube End Caps
- Decorative Electronic Components
    - Capacitors
    - Resistors
    - PDIP Chip

# Exterior Design

First I cut the aluminum sheets and perfboard to size. I used a ruler and a sharp utility knife
for both, scoring until I could snap them.

![Exterior Parts](/assets/divmeter/exterior-parts.jpg)
*Exterior Parts Cut Out*

Next I drilled holes in the perfboard for the nixie tubes to slot in. My first attempt (pictured on bottom)
wasn't great, so I redid it.

![Drilled Perfboard](/assets/divmeter/perfboard-cuts.jpg)
*Drilled Perfboard*

I drilled holes in the back aluminum panel for the two control buttons, power jack, and power switch.
To achieve a rectangular hole for the power switch, I first drilled some smaller circular holes, then
filed the edges down to a rectangle.

![Back Buttons](/assets/divmeter/buttons.jpg)
*Installed Buttons*

I taped together the panels to hold them steady. Then, I applied epoxy to the inner corners to join them.
The long hex nuts were also epoxied to the inner corners, allowing the top perfboard panel to be securely screwed
in to the rest of the body.

![Back Buttons](/assets/divmeter/partially-assembled-exterior.jpg)
*Partially Assembled Exterior*

I modeled and 3D printed some end caps for the nixie tubes so they would sit flat.
I also glued dummy electronic components to the perfboard, matching the show's design.

![Decorative Electronic Components](/assets/divmeter/decorative-electronics.jpg)
*Decorative Electronic Components*

Finally, I glued plastic corners to the outer edges and screwed on the bottom perfboard and plastic panels.

![Finished Exterior](/assets/divmeter/finished-exterior.jpg)
*Finished Exterior*

# Electronics

I designed two PCBS: a Logic Board, containing the microcontroller, RTC chip, HV power supply,
and associated parts, and a Nixie Board, containing HV serial to parallel converters, pulldown
resistors, and solder points for the tubes. The two boards are connected with 8 pins that transfer
ground, HV, and data.

![Logic Board](/assets/divmeter/div-logic-board.jpg)
*Logic Board*

![Divergence Meter](/assets/divmeter/div-nixie-board.jpg)
*Nixie Board*

After soldering everything together and connecting the boards, it was time to write some software!

# Software

I used MPLAB IDE X to write the software in C and compile it down to hex.
I then flashed it onto the PIC with my Arduino UNO and a command line tool called <a href="https://github.com/battlecoder/zeppp">ZEPPP</a>.
The PIC16F628A only has 2048 bytes of program memory, so I had to be very intentional with my coding.
The first step was getting digits to display on the tubes.

The way I decided to implement tube control was having a `digits` buffer, which stores the 0-9 value to display for
each digit (plus 10 signifying decimal and 11 signifying off).
The `bits` buffer stores the final bits we are going to write to the serial to parallel converters.

Due to the way I designed the connections from the tubes to the serial to parallel converters in the PCB,
I had to do some bit shuffling to get the serial output in the correct order. This seemed unavoidable
if I wanted nice PCB wiring. I originally hardcoded a series of conditionals to produce the correct mapping,
but I managed to figure out a more compact formula, which saved program space at the cost of a little working data space.

```c
char bits[12];
char digits[8];
char tubeBitIndices[] = { 8, 9, 10, 4, 5, 6, 1, 2 };
char tubeBitOffsets[] = { 8, 5, 2, 8, 5, 2, 5, 2 };

// This function converts the desired digits to the correct bit output
void GetTubeBits()
{
    // If tube digit 1 is to be set to 1, the bits for that would be 00000000010
    // Each Tube has 10 bits corresponding to it and 1 extra NC bit (always zero)
    // 11 * 8 = 88 bits + 8 = 96
    // Tube 3 corresponds to indices 0 to 10 (from the end)
    // Tube 2 corresponds to indices 11 to 21 (from the end)
    // Tube 1 corresponds to indices 22 to 31 (from the end)
    // Tube 6 corresponds to indices 32 to 42 (from the end)
    // Tube 5 corresponds to indices 43 to 53(from the end)
    // Tube 4 corresponds to indices 54 to 63 (from the end)
    // Tube 8 corresponds to indices 64 to 74 (from the end)
    // Tube 7 corresponds to indices 75 to 85 (from the end)
    // Decimals correspond to indices 86 to 95 (from the end)
    
    // 0 8 16 24 32 40 48 56 64 72 80 88 96
    int temp;
    int mask;
    for(char i = 0; i < 8; i++)
    {
        if(digits[i] < 10)
        {
            mask = 0b1111111111111111 >> (16 - tubeBitOffsets[i]);
            temp = 1 << ((digits[i] + 9) % 10);
            bits[tubeBitIndices[i]] |= (char) ((temp & mask) << (8 - tubeBitOffsets[i]));
            bits[tubeBitIndices[i] + 1] |= (char) ((temp & (~mask & 0b0000001111111111)) >> tubeBitOffsets[i]);
        }
        else if(digitsFinal[i] == 10)
        {
            bits[i / 6] |= (1 << ((i + 2) % 8));
        }
    }
}
```

Now all I have to do is populate the digits buffer with the digits I want displayed,
and call the DisplayDigits function, which will write the correct bits to the serial to parallel
converters and finally light the correct tube digits.

```c
void DisplayDigits()
{
    for(char i = 0; i < 12; i++) bits[i] = 0;

    GetTubeBits();

    // Clock pin:
    RB1 = 1;
    for(char i = 0; i < 12; i++)
    {
        // Send bits:
        for(char j = 0; j < 8; j++)
        {
            RA3 = (bits[i] & (1 << j)) >> j;
            RB1 = 0;
            RB1 = 1;
        }
    }

    // Unlatch:
    RA0 = 1;
    RA0 = 0;
}
```

![First Test](/assets/divmeter/first-test.jpg)
*Testing the Tubes*

## Connecting to the RTC Chip

Since we want to be able to use this as a clock, we need to connect to the RTC chip to read the current time.
The DS3232SN# uses the I2C communication protocol for serial data transfer.
After writing some boiler plate code and referencing the datasheet, we can successfully connect to the RTC!
Reading the time is as simple as sending the correct register address to the chip.

![First Test](/assets/divmeter/address-map.png)
*DS3232 Address Map*

We can now read the time!

<video width="100%" autoplay muted loop playsinline>
    <source src="/assets/divmeter/divclock.mp4" type="video/mp4">
</video>

We can set the time by writing to the registers instead of reading.
Notice that addresses 14h-0FFh are listed as SRAM. This is general purpose, battery backed memory we can use 
to store persistent data. Right now I have it store the current worldline and its expiration date. 

The rest of the coding process was adding features like button interaction, menus, visual effects, and SRAM debug read/write.

# Finished Product

Here are some shots of the finished product.

![finished-clock](/assets/divmeter/finished-clock.jpg)

![finished-clock](/assets/divmeter/finished-clock-2.jpg)

<video width="100%" muted controls loop playsinline>
    <source src="/assets/divmeter/day-clock.mp4" type="video/mp4">
</video>

<video width="100%" muted controls loop playsinline>
    <source src="/assets/divmeter/night-clock.mp4" type="video/mp4">
</video>

<video width="100%" muted controls loop playsinline>
    <source src="/assets/divmeter/scramble-effect.mp4" type="video/mp4">
</video>

# Final Thoughts

This was a super fun project and I definitely learned a lot. The clock works perfectly and has not lost so much as a second
after a year of operation. I have it so that every day at midnight, a new world line number is generated. This is then 
displayed at random times throughout the day. Some software features I still want to implement include an anti-poisoning
routine for the tubes, more effects, dimming, and a date display. If I were to design it from scratch again, I would try
to condense the internals even more to make room for a rechargeable battery. Maybe I'll revisit it in a different world line...