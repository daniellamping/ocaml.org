---
title: OCaml Clock on Pi Pico 2W
description: After playing with the Pi Pico 2W at the New Year, I had a little time
  today and made an OCaml-powered clock in a 3D-printed case.
url: https://www.tunbury.org/2026/04/06/pico-clock/
date: 2026-04-06T21:22:00-00:00
preview_image: https://www.tunbury.org/images/pico-clock-cad.png
authors:
- Mark Elvers
source:
ignore:
---

<p>After playing with the <a href="https://www.tunbury.org/2025/12/31/ocaml-pico/">Pi Pico 2W</a> at the New Year, I had a little time today and made an OCaml-powered clock in a 3D-printed case.</p>

<p>It’s overcomplicated; I have two cores available, and I really wanted to use both of them, so core 0 handles the NTP sync, leaving core 1 to handle the display refresh. The code is written in OCaml 5 using my ARM 32 native code <a href="http://www.tunbury.org/2025/11/27/ocaml-54-native/">backend</a>.</p>

<p><img src="https://www.tunbury.org/images/pico-clock-front.png" alt="Pi Pico front view"></p>

<p>Here’s my pinout:</p>

<table>
  <thead>
    <tr>
      <th>Pi Pico</th>
      <th>Label</th>
      <th>LCD Pin</th>
      <th>Label</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>38</td>
      <td>GND</td>
      <td>1</td>
      <td>VSS</td>
    </tr>
    <tr>
      <td>40</td>
      <td>VBUS 5V</td>
      <td>2</td>
      <td>VDD</td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>3</td>
      <td>VO</td>
    </tr>
    <tr>
      <td>21</td>
      <td>GP16</td>
      <td>4</td>
      <td>RS</td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>5</td>
      <td>RW -&gt; VDD</td>
    </tr>
    <tr>
      <td>22</td>
      <td>GP17</td>
      <td>6</td>
      <td>E</td>
    </tr>
    <tr>
      <td>24</td>
      <td>GP18</td>
      <td>11</td>
      <td>D4</td>
    </tr>
    <tr>
      <td>25</td>
      <td>GP19</td>
      <td>12</td>
      <td>D5</td>
    </tr>
    <tr>
      <td>26</td>
      <td>GP20</td>
      <td>13</td>
      <td>D6</td>
    </tr>
    <tr>
      <td>27</td>
      <td>GP21</td>
      <td>14</td>
      <td>D7</td>
    </tr>
    <tr>
      <td>29</td>
      <td>GP22</td>
      <td>15</td>
      <td>K</td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>16</td>
      <td>A -&gt; VDD</td>
    </tr>
  </tbody>
</table>

<ul>
  <li>VO is connected to the centre tap of 100K potentiometer</li>
</ul>

<p><img src="https://www.tunbury.org/images/pico-clock-rear.png" alt="Pi Pico rear view"></p>

<p>The LCD 2004 is the kind without the I2C backpack. I have used four GPIO lines driving the HD44780 in 4-bit mode for easier wiring.</p>

<p>The backlight is controlled by PWM pin 22 on the Pico, allowing it to be dimmed at night.</p>

<iframe width="560" height="315" src="https://www.youtube.com/embed/_rSSekcB6w8?si=kEyOaHCXxEsfddTA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen=""></iframe>

<p>I’ll post my post in the next few days once I have tidied it up a bit.</p>
