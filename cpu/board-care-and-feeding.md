# Care and feeding for your C5G board#

## Tools ##

If you wish to develop for the FPGA itself, you probably want a copy of
Quartus II installed.  Quartus II Web Edition will do fine; it runs on Linux
and Windows.  You can get it from: http://dl.altera.com/?edition=web .

If you are on a disk space budget, you can avoid downloading the whole thing
by downloading only the Quartus II base package, and the Cyclone V device
support.  (We haven't decided on a simulator yet.) If you only intend to
flash your board, and you don't want to use OpenOCD to do it (or you want to
actually program the on-board flash device), you can download the Quartus II
Programmer from 'Additional Software', and you don't need the rest of the
package.  217MB for a glorified JTAG tool!  What a bargain!

If you're on Linux, you probably need to udev yourself to get permissions to
access the board.  To do this, drop the following in
`/etc/udev/rules.d/40-usbblaster.rules`:

   SUBSYSTEM=="usb", ATTRS{idVendor}=="09fb", ATTRS{idProduct}=="6001", GROUP="plugdev", MODE="0666", SYMLINK+="usbblaster"

## How to boot your board ##

### ...without Quartus installed ###

If you don't have the Quartus tools, get a copy of OpenOCD 0.8.0, and patch it with the following patch: http://openocd.zylin.com/2145 .  Then, configure with `--enable-usb_blaster_libftdi` (make sure you have libftdi installed on your system).

To program your board with a premade .SVF file, use a command like: `openocd -c "interface usb_blaster" -c "jtag newtap auto0 tap -expected-id 0x02b020dd -irlen 10" -c "init" -c "svf $1" -c "shutdown"`.  Make sure your board is set to RUN before running this command (nothing bad will happen if it's in PROG, but it just won't work).

### ...with Quartus installed ###

To program a .sof file to your board, you can use `quartus_pgm -m JTAG -o p\;$1`.  (Thanks, Altera, for having a semicolon as part
of your required command syntax.) The switch should be set to RUN for this
operation.

## How to flash an image to the on-board Flash device ##

If you want a certain FPGA bitstream to boot every time you power on the
board, you might want to flash it to the attached flash device.  To do this,
set the switch on your board to PROG, instead of RUN.  In theory, it should
be possible to cook up .svf files to do this that you can feed to OpenOCD,
but in practice, I haven't had any luck doing that yet.

First, convert the .sof (bitstream format) file to a .pof (flash format) file.  To do this, use: `quartus_cpf --convert -dEPCQ256 $PROJ.sof $PROJ.pof

Then, to program the .pof, you can use `quartus_pgm -m AS -o p\;$PROJ.pof`.  (On my machine, this takes about two and a half minutes; don't worry.)
