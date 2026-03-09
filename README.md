# Color picker for Philips Hue

This repository is the very beginning of a CIE 1931 x/y
color picker for Philips Hue lights. It is to be used
in Node Red using Dashboard 2.0.

I just started this project. There is no usable code yet.
This is work in progress.

The problem with existing color pickers is the fact that
they are designed for the RGB color space, while Philips Hue
operates in the XY color space. The API for Philips Hue has
limited support for RGB, however it does so by converting
RGB values into XY values and vice versa, causing information
loss, which may lead to strange results.
For example, setting the color to (0, 0, 255) will result
in color (64, 0, 255). Feeding that back into the color picker
will result in the cursor jumping far away from the user
selected color. This color picker attempts to address
this problem, by operating in the XY color space directly.