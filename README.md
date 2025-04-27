# Milling a PCB with auto-leveling using a Carbide 3D Nomad 3

I tried everything between [Carbide Copper](https://carbide3d.com/copper/), which seems abandoned at best and buggy at worst: it does not seem to spin the spindle. I played with [FlatCAM](http://flatcam.org/) and [bCNC](https://github.com/vlachoudis/bCNC), which I used to brick my Nomad 3. (I provide [instructions to reset the GRBL on a Nomad 3](https://github.com/thomergil/carbide3d-grbl-recovery?tab=readme-ov-file), much thanks to Carbide 3D support.)

Ultimately, I settled on [pcb2gcode](https://github.com/pcb2gcode/pcb2gcode) and [OpenCNCPilot](https://github.com/martin2250/OpenCNCPilot), which supports auto-leveling. Unfortunately–I typically use Mac and Linux–**OpenCNCPilot only works on Windows**. Annoying as that may be, it might be worth the sacrifice, in this case. You can operate your Nomad 3 fully using Windows.

# WORK IN PROGRESS

**This document is being (re)written and edited in April, early May 2025.**

# Acknowledgements

This has been partly inspired by [Chris Kohlhardt's notes](https://www.chriskohlhardt.com/small-double-sided-pcb-traces-on-nomad-cnc).

# CNC Machine

This has been tested on a [Carbide 3D Nomad 3](https://shop.carbide3d.com/products/nomad-3) CNC Mill.

# Mac? Windows? Linux?

Unfortunately, [OpenCNCPilot](https://github.com/martin2250/OpenCNCPilot) exists only for Windows. 🙁  It needs to be attached directly to the CNC Machine. I have not (yet) tried running it using a VM.

# PCB blanks / Copper Clad

I use 2x3" and 4x6" PCB blanks that I order [from Carbide 3D](https://shop.carbide3d.com/collections/materials/products/fr1-copper-clad?variant=41237063046). They come single-sided ("SS") or double-sided ("DS"). They are also sold on Amazon and AliExpress.

# Drill/Mill bits

30º Vee bits:

* [#501 PCB Engraver (30º) bits from Carbide 3D](https://shop.carbide3d.com/products/501-engraving-bit?_pos=3&_psq=pcb&_ss=e&_v=1.0) 
* [Coated 30º V-Groove bits from bits&bits](https://bitsbits.com/product/60-degree-v-groove/)
* [FoxAlien 30º bits via Amazon](https://a.co/d/37hDOj5)

60º Vee bits:

* [#502 PCB Engraver (40º) bits from Carbide3D](https://shop.carbide3d.com/products/502-engraving-bit?_pos=2&_psq=pcb&_ss=e&_v=1.0)
* [Coated 60º V-Groove bits from bits&bits](https://bitsbits.com/product/425-vg30/)
* [FoxAlien 60º bits via Amazon](https://www.amazon.com/dp/B08881PKBN?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1)

# Jigs

I provide several different jigs on http://printables.com.

* A jig for drilling and etching a 2x3" copper clad.
* A jig just for drilling a 2x3" copper clad.
* A jig just for drilling a 4x6" copper clad.
* A jig for etching a 2x3" and 4x6" copper clad.

If you don't have access to a 3D printer, contact me, and I can make them for you if time allows. (Large ones are $49.99, and small ones are $39.99 plus shipping.)

# BitZero attachment

You need to rig the [BitZero V2](https://shop.carbide3d.com/products/bitzero-v2-for-nomad-3). Anything works as long as it conducts ground between the top of the copper clad and the BitZero. This is my solution: 

![bitzero-rig](bitzero-rig1.png)

The BitZero is squeezed on both sides by two metal rings, held in place by a bolt and screw; an alligator clip attaches to the bolt. The alligator clip connects to a ring terminal that attaches to the copper clad as follows:

![bitzero-rig](bitzero-rig2.png)

# KiCad PCB Editor

A KiCad tutorial is outside this document's scope, but a few things to keep in mind when exporting from KiCad's PCB editor:

* Make the Edge Cut layer match exactly the dimensions of the PCB blank.
* If you are following this tutorial, there will be screws in the corners, keep your circuitry plenty far away from the corners.
* Ensure a few millimeters of space between the edges and the circuitry.
* Using menu Place → Place Drill/Place File Origin, place the origin at the bottom left of the Edge Cut layer.
* Similarly, using menu Place → Grid Origin, place the origin at the bottom left of the Edge Cut layer.

# KiCad Plot / Export

Using menu File → Plot...:

* :heavy_check_mark: F. Cu
* :heavy_check_mark: B. Cu
* :heavy_check_mark: Edge.Cuts
* :heavy_check_mark: Use drill/place file origin
* :x: Use extended X2 format (meaning, make sure this option is unchecked)

Then:

* Generate Drill Files...
* Excellon format
* Origin: Drill/place file origin
* Units: millimeters
* Zeros: decimal format (recommended)

Then press "Generate".

Back on the Plot window, press "Plot"

# Generate Gcode from .gbr files

Create a file `millproject` in the same directory as the `.gbr` files.

```
metric=true
metricoutput=true
zwork=-0.05          # Depth in mm,
zsafe=20             # Height for movements
zchange=55           # Height for tool changes
mill-feed=100        # Feed rate in mm/minute
mill-speed=12000     # RPM
nom6=1               # Don't issue M6 command
spinup-time=3.0      # Time to spin up the spindle in seconds
spindown-time=3.0    # Time to spin up the spindle in seconds
backtrack=0          # Don't criss-cross copper; always travel at zsafe

# this will mess up offset! don't use!
# zero-start=true
```

Some of these values are **critically important**:

* `zsafe` is the travel height of the mill bit; **if this is too low your bit will crash into the jig or screws, breaking the mill or worse**; I recommend at least 20 (mm), but a higher value is fine. It will be slower but safer.
* `zwork` is how deep the mill drills into the copper substrate; start with `0.025` (for a 60º Vbit) or `0.03` and iterate deeper as needed; this also depends on the Vee bit you are using.
* `nom6=1` prevents `pcb2gcode` from issuing an `M6` command, which trips up the Nomad 3. 

Run [pcb2gcode](https://github.com/pcb2gcode/pcb2gcode):

```bash
# replace "decibel-metel-F_Cu.gbr" with your F_Cu file
# --basename is optional, but without it, the output will be "front.ngc"
pcb2gcode --front decibel-meter-F_Cu.gbr --basename db-LED
```

This creates a `.ngc` file.

# OpenCNCPilot

OpenCNCPilot is a little quirky, but it does everything you need it to. Please start by reading through the [OpenCNCPilot README](https://github.com/martin2250/OpenCNCPilot/blob/master/README.md).

### General notes on OpenCNCPilot

* As you are learning OpenCNCPilot, **stay close to your CNC machine with your finger on the power switch**.
* There is not enough vertical space to fold open all menus, so you need to manage them. You'll use **File**, and **Edit**, and **Probing**, and **Manual**, and **Manual Probing**. I try to only ever open one at the same time.
* I believe you cannot reposition the view, but you can zoom in and zoom out and use it to "direct" the view. Under the **Debug** menu (on the right) you can **Lay flat 3D Viewport** and **Restore Viewport**.
* I have destroyed a half dozen mill bits while learning OpenCNCPilot. I suggest you start with cheap throwaway bits while you're learning.
* You can home to (X = 0, Y = 0) without changing height by typing the command `G0 X0 Y0` in the **Manual** menu and then pressing **Send**.

### OpenCNCPilot and Carbide Motion don't play nice

Switching **to** OpenCNCPilot after using Carbide Motion is usually OK. However, using Carbide Motion **after** OpenCNCPilot can fail in mysterious ways, starting with initialization and homing. Stay close and keep your finger on the power button. If weird things happen, follow my [instructions to reset the GRBL on a Nomad 3](https://github.com/thomergil/carbide3d-grbl-recovery?tab=readme-ov-file).