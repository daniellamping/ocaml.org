---
title: OCaml on a Raspberry Pi
description: The weather outside is frightful, but the Raspberry Pi is so delightful;
  I have been cheering myself by connecting up all the various bits of hardware scattered
  on my desk. I often buy these components but never quite get around to using them.
url: https://www.tunbury.org/2025/11/15/ocaml-raspberry-pi/
date: 2025-11-15T22:00:00-00:00
preview_image: https://www.tunbury.org/images/raspberry-pi-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>The weather outside is frightful, but the Raspberry Pi is so delightful; I have been cheering myself by connecting up all the various bits of hardware scattered on my desk. I often buy these components but never quite get around to using them.</p>

<p>My latest purchase was the <a href="https://www.amazon.co.uk/dp/B07J3FHJVP">Waveshare 2.13” e-Paper Display HAT</a>, which is exactly the same size as a Pi Zero. The basic interface is SPI, plus the device uses various GPIO lines. The drivers provided are in C and Python, and unsurprisingly, no OCaml. Looking on opam, there is <a href="https://opam.ocaml.org/packages/wiringpi/">wiringpi</a>, which provides OCaml bindings for the WiringPi library for OCaml &lt; 5.0.</p>

<p>Do I need a 3rd party library? The kernel provides <code class="language-plaintext highlighter-rouge">/dev/spi*</code> and <code class="language-plaintext highlighter-rouge">/dev/i2c*</code> when these interfaces are enabled with <code class="language-plaintext highlighter-rouge">raspi-config</code>. GPIO can be accessed via <code class="language-plaintext highlighter-rouge">/sys/bus/gpio</code>, but this interface is deprecated and only provides a subset of the full functionality. All I really need to do is call <code class="language-plaintext highlighter-rouge">ioctl()</code> on <code class="language-plaintext highlighter-rouge">/dev/gpiochipN</code>, and I can access that via Ctypes.</p>

<p>Experimenting with some basic functionality, I managed to blink an LED on GPIO17.</p>

<p><img src="https://www.tunbury.org/images/gpio-led.jpg" alt=""></p>

<p>After that, I was hooked. Adding I2C to read from a <a href="https://www.amazon.co.uk/WINGONEER-DS3231-AT24C32-Precision-Arduino/dp/B01H5NAFUY">DS3231 real time clock with EEPROM</a>, followed by SPI to output to an <a href="https://www.amazon.co.uk/MAX7219-Matrix-Display-Arduino-Microcontroller/dp/B07YWRZ3FC">LED matrix</a>.</p>

<p><img src="https://www.tunbury.org/images/gpio-max7219.jpg" alt=""></p>

<p>I found a large LCD2004 display with an I2C driver board, so that was my next target. These are handy displays for basic text. They limit you to 8 custom characters, but if you think about it, a seven-segment display only needs seven elements so you can turn that into a nice big retro digital clock!</p>

<p><img src="https://www.tunbury.org/images/gpio-lcd2004.jpg" alt=""></p>

<p>On to the e-Paper display and basic framebuffer display. This display is very cool as it has two buffers and can do a partial update of the display from the secondary buffer without needing to refresh the display completely.</p>

<p><img src="https://www.tunbury.org/images/gpio-epaper.jpg" alt=""></p>

<p>The library and test code are available at <a href="https://github.com/mtelvers/gpio">mtelvers/gpio</a>.</p>
