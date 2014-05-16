# Care and feeding for your C5G board#

## How to boot your board ##

If you don't have the Quartus tools, get a copy of OpenOCD 0.8.0, and patch it with the following patch: http://openocd.zylin.com/2145 .  Then, configure with `--enable-usb_blaster_libftdi --enable-usb_blaster_libftd2xx`.

To program your board with a premade .SVF file, use a command like: `openocd -c "interface usb_blaster" -c "jtag newtap auto0 tap -expected-id 0x02b020dd -irlen 10" -c "init" -c "svf $1" -c "shutdown"`.  Make sure your board is set to RUN before running this command (nothing bad will happen if it's in PROG, but it just won't work).
