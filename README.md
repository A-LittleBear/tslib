

![logo](website/tslib_logo_web.jpg?raw=true)
[![Coverity Scan Build Status](https://scan.coverity.com/projects/11027/badge.svg)](https://scan.coverity.com/projects/tslib)
[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/752/badge)](https://bestpractices.coreinfrastructure.org/projects/752)

# C library for filtering touchscreen events

tslib consists of the library _libts_ and tools that help you _calibrate_ and
_use it_ in your environment. There's a [short introductory presentation from 2017](https://fosdem.org/2017/schedule/event/tslib/).

## contact
If you have problems, questions, ideas or suggestions, please contact us by
posting to [our mailing list](http://lists.infradead.org/mailman/listinfo/tslib).

## website
Visit the [tslib website](http://tslib.org) for an overview of the project.

## table of contents
* [setup and configure tslib](#setup-and-configure-tslib)
* [filter modules](#filter-modules)
* [the libts library](#the-libts-library)
* [building tslib](#building-tslib)


## setup and configure tslib
### install tslib
tslib runs on various operating systems, including GNU/Linux,
FreeBSD or Android/Linux. See [building tslib](#building-tslib) for details.
Apart from building the latest tarball release, running
`./configure`, `make` and `make install`, tslib is available from the following
distributors and their package management:
* [Arch Linux](https://www.archlinux.org) and [Arch Linux ARM](https://archlinuxarm.org) - `pacman -S tslib`
* [Buildroot](https://buildroot.org/) - `BR2_PACKAGE_TSLIB=y`
* (Debian: [Looking for a sponsor](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=854693) to include it)

### set up environment variables
    TSLIB_TSDEVICE          TS device file name.
                            Default (inputapi):     /dev/input/ts
                            /dev/input/touchscreen
                            /dev/input/event0
                            Default (non inputapi): /dev/touchscreen/ucb1x00
    TSLIB_CALIBFILE         Calibration file.
                            Default: ${sysconfdir}/pointercal
    TSLIB_CONFFILE          Config file.
                            Default: ${sysconfdir}/ts.conf
    TSLIB_PLUGINDIR         Plugin directory.
                            Default: ${datadir}/plugins
    TSLIB_CONSOLEDEVICE     Console device.
                            Default: /dev/tty
    TSLIB_FBDEVICE          Framebuffer device.
                            Default: /dev/fb0

* On Debian, `TSLIB_PLUGINDIR` probably is for example `/usr/lib/x86_64-linux-gnu/ts0`.
* Find your `/dev/input/eventX` touchscreen's event file and either
  - Symlink `ln -s /dev/input/eventX /dev/input/ts` or
  - `export TSLIB_TSDEVICE /dev/input/eventX`
* If you are not using `/dev/fb0`, be sure to set `TSLIB_FBDEVICE`

### configure tslib
This is just an example `/etc/ts.conf` file. Touch samples flow from top to
bottom. Each line specifies one module and it's parameters. Modules are
processed in order. Use _one_ module_raw that accesses your device, followed
by any combination of filter modules.

    module_raw input
    module median depth=3
    module dejitter delta=100
    module linear

see the [section below](#filter-modules) for available filters and their
parameters.

With this configuration file, we end up with the following data flow
through the library:

    driver --> raw read --> median  --> dejitter --> linear --> application
               module       module      module       module

### calibrate the touch screen
Calibration is done by the `linear` plugin, which uses it's own config file
`/etc/pointercal`. Don't edit this file manually. It is created by the
`ts_calibrate` program:

    # ts_calibrate

The calibration procedure simply requires you to touch the cross on screen,
where is appears, as accurate as possible.

### test the filtered input behaviour
You may quickly test the touch behaviour that results from the configured
filters, using `ts_test_mt`:

    # ts_test_mt

### use the filtered result in your system
You need a tool using tslib'd API and provide it to your input system. There are
various ways to do so on various systems. We only describe one way for Linux
here - using tslib's included userspace input evdev driver `ts_uinput`:

    # ts_uinput -d -v

`-d` makes the program return and run as a daemon in the background. `-v` make
it print the new `/dev/input/eventX` device node before returning.

In this case, for Qt5 for example you'd probably set something like this:

    QT_QPA_GENERIC_PLUGINS=evdevtouch:/dev/input/eventX
    QT_QPA_EVDEV_TOUCHSCREEN_PARAMETERS=/dev/input/eventX:rotate=0

For X11 you'd probably edit your `xorg.conf` `Section "InputDevice"` for your
touchscreen to have

    Option "Device" "/dev/input/eventX"

and so on. Please see your system's documentation on how to use a specific
evdev input device.

Remember to set your environment and configuration for ts_uinput, just like you
did for ts_calibrate or ts_test_mt.

Let's recap the data flow here:

    driver --> raw read --> filter --> (...)   --> ts_uinput  --> evdev read
               module       module     module(s)   application    application


## filter modules

### module: linear

#### Description:
  Linear scaling - calibration - module, primerily used for conversion of touch
  screen co-ordinates to screen co-ordinates. It applies the corrections as
  recorded and saved by the `ts_calibrate` tool. It's the only module that reads
  a configuration file.

#### Parameters:
* `xyswap`

	interchange the X and Y co-ordinates -- no longer used or needed
	if the linear calibration utility `ts_calibrate` is used.

* `pressure_offset`

	offset applied to the pressure value
* `pressure_mul`

	factor to multiply the pressure value with
* `pressure_div`

	value to divide the pressure value by


### module: median

#### Description:
  Similar to what a variance filter does, the median filter suppresses
  spikes in the gesture. For some theory, see [Wikipedia](https://en.wikipedia.org/wiki/Median_filter)

Parameters:
* `depth`

	Number of samples to apply the median filter to


### module: pthres

#### Description:
  Pressure threshold filter. Given a release is always pressure 0 and a
  press is always >= 1, this discards samples below / above the specified
  pressure threshold.

#### Parameters:
* `pmin`

	Minimum pressure value for a sample to be valid.
* `pmax`

	Maximum pressure value for a sample to be valid.


### module: iir

#### Description:
  Infinite impulse response filter. Similar to dejitter, this is a smoothing
  filter to remove low-level noise. There is a trade-off between noise removal
  (smoothing) and responsiveness. The parameters N and D specify the level of
  smoothing in the form of a fraction (N/D).

  [Wikipedia](https://en.wikipedia.org/wiki/Infinite_impulse_response) has some
  general theory.


Parameters:
* `N`

	numerator of the smoothing fraction
* `D`

	denominator of the smoothing fraction


### module: dejitter

#### Description:
  Removes jitter on the X and Y co-ordinates. This is achieved by applying a
  weighted smoothing filter. The latest samples have most weight; earlier
  samples have less weight. This allows to achieve 1:1 input->output rate. See
  [Wikipedia](https://en.wikipedia.org/wiki/Jitter#Mitigation) for some general
  theory.

#### Parameters:
* `delta`

	Squared distance between two samples ((X2-X1)^2 + (Y2-Y1)^2) that
	defines the 'quick motion' threshold. If the pen moves quick, it
	is not feasible to smooth pen motion, besides quick motion is not
	precise anyway; so if quick motion is detected the module just
	discards the backlog and simply copies input to output.


### module: debounce

#### Description:
  Simple debounce mechanism that drops input events for the specified time
  after a touch gesture stopped. [Wikipedia](https://en.wikipedia.org/wiki/Switch#Contact_bounce)
  has more theory.

#### Parameters:
* `drop_threshold`

	drop events up to this number of milliseconds after the last
	release event.


### module: skip

#### Description:
  Skip nhead samples after press and ntail samples before release. This
  should help if for the device the first or last samples are unreliable.

  Note that it is still **experimental for multitouch**.

Parameters:
* `nhead`

	Number of events to drop after pressure
* `ntail`

	Number of events to drop before release


### module:	variance

#### Description:
  Variance filter. Tries to do it's best in order to filter out random noise
  coming from touchscreen ADC's. This is achieved by limiting the sample
  movement speed to some value (e.g. the pen is not supposed to move quicker
  than some threshold).

  This is a 'greedy' filter, e.g. it gives less samples on output than
  receives on input. It can cause problems on capacitive touchscreens that
  already apply such a filter.

  There is **no multitouch** support for this filter (yet). `ts_read_mt()` will
  only read one slot when this filter is used. You can try using the median
  filter instead.

#### Parameters:
* `delta`

	Set the squared distance in touchscreen units between previous and
	current pen position (e.g. (X2-X1)^2 + (Y2-Y1)^2). This defines the
	criteria for determining whenever two samples are 'near' or 'far'
	to each other.

	Now if the distance between previous and current sample is 'far',
	the sample is marked as 'potential noise'. This doesn't mean yet
	that it will be discarded; if the next reading will be close to it,
	this will be considered just a regular 'quick motion' event, and it
	will sneak to the next layer. Also, if the sample after the
	'potential noise' is 'far' from both previously discussed samples,
	this is also considered a 'quick motion' event and the sample sneaks
	into the output stream.



***
The following example setup

           |--------|       |-----|      |--------------|
    x ---> | median | ----> | IIR | ---> |              | ---> x'
           |--------|    -> |-----|      |    screen    |
                        |                |  transform   |
                        |                | (calibrate)  |
           |--------|   |   |-----|      |              |
    y ---> | median | ----> | IIR | ---> |              | ---> y'
           |--------|   |-> |-----|      |--------------|
                        |
                        |
                 |----------|
    p ---------> | debounce | -------------------------------> p'
                 |----------|

would be achieved by the following ts.conf:

    module_raw input
    module debounce drop_threshold=40
    module median depth=5
    module iir N=6 D=10
    module linear

while you are free to play with the parameter values.

***


## the libts library
### the libts API
Check out our tests directory for examples how to use it.

`ts_open()`  
`ts_config()`  
`ts_setup()`  
`ts_close()`  
`ts_reconfig()`  
`ts_option()`  
`ts_fd()`  
`ts_load_module()`  
`ts_read()`  
`ts_read_raw()`  
`ts_read_mt()`  
`ts_reat_raw_mt()`  

The API is documented in our man pages in the doc directory.
Possibly there will be distributors who provide them online, like
[Debian had done for tslib-1.0](https://manpages.debian.org/wheezy/libts-bin/index.html).
As soon as there are up-to-date html pages hosted somewhere, we'll link the
functions above to it.

### ABI - Application Binary Interface

[Wikipedia](https://en.wikipedia.org/wiki/Application_binary_interface) has
background information.

#### libts Soname versions
Usually, and every time until now, libts does not break the ABI and your
application can continue using libts after upgrading. Specifically this is
indicated by the libts library version's major number, which should always stay
the same. According to our versioning scheme, the major number is incremented
only if we break backwards compatibility. The second or third minor version will
increase with releases. In the following example


    libts.so -> libts.so.0.3.1
    libts.so.0 -> libts.so.0.3.1
    libts.so.0.3.1


use `libts.so` for using tslib unconditionally and `libts.so.0` to make sure
your current application never breaks.

If a release includes changes like added features, the second number is
incremented and the third is set to zero. If a release includes mostly just
bugfixes, only the third number is incremented.

#### tslib package version
A tslib tarball version number doesn't tell you anything about it's backwards
compatibility.

### dependencies

* libc (with libdl if you build it dynamically linked)

### related libraries

* [libevdev](https://www.freedesktop.org/wiki/Software/libevdev/) - access wrapper for [event device](https://en.wikipedia.org/wiki/Evdev) access, uinput too ("Linux API")
* [libinput](https://www.freedesktop.org/wiki/Software/libinput/) - handle input devices for Wayland (uses libevdev)
* [xf86-input-evdev](https://cgit.freedesktop.org/xorg/driver/xf86-input-evdev/) - evdev plugin for X (uses libevdev)

### libts users

* ts_uinput - userspace event device driver for the tslib-filtered samples. part of tslib (tools/ts_uinput.c)
* [xf86-input-tslib](http://public.pengutronix.de/software/xf86-input-tslib/) - **outdated** direct tslib input plugin for X
* [qtslib](https://github.com/qt/qtbase/tree/dev/src/platformsupport/input/tslib) - **outdated** Qt5 qtbase tslib plugin

### using libts
This is a complete example program, similar to `ts_print_mt.c`:

    #include <stdio.h>
    #include <stdlib.h>
    #include <fcntl.h>
    #include <sys/time.h>
    #include <unistd.h>

    #include "tslib.h"

    #define SLOTS 5
    #define SAMPLES 1

    int main(int argc, char **argv)
    {
        struct tsdev *ts;
        char *tsdevice = NULL;
        struct ts_sample_mt **samp_mt = NULL;
        struct input_absinfo slot;
        unsigned short max_slots = 1;
        unsigned short read_samples = 1;
        int ret, i, j;

        ts = ts_setup(tsdevice, 0);
        if (!ts) {
                perror("ts_setup");
                return -1;
        }

        max_slots = SLOTS;
        read_samples = SAMPLES;

        samp_mt = malloc(read_samples * sizeof(struct ts_sample_mt *));
        if (!samp_mt) {
                ts_close(ts);
                return -ENOMEM;
        }
        for (i = 0; i < read_samples; i++) {
                samp_mt[i] = calloc(max_slots, sizeof(struct ts_sample_mt));
                if (!samp_mt[i]) {
                        free(samp_mt);
                        ts_close(ts);
                        return -ENOMEM;
                }
        }

        while (1) {
                ret = ts_read_mt(ts, samp_mt, max_slots, read_samples);
                if (ret < 0) {
                        perror("ts_read_mt");
                        ts_close(ts);
                        exit(1);
                }

                for (j = 0; j < ret; j++) {
                	for (i = 0; i < max_slots; i++) {
				if (samp_mt[j][i].valid != 1)
					continue;

				printf("%ld.%06ld: (slot %d) %6d %6d %6d\n",
				       samp_mt[j][i].tv.tv_sec,
				       samp_mt[j][i].tv.tv_usec,
				       samp_mt[j][i].slot,
				       samp_mt[j][i].x,
				       samp_mt[j][i].y,
				       samp_mt[j][i].pressure);
                        }
                }
        }

        ts_close(ts);
    }


If you know how many slots your device can handle, you could avoid malloc:

    struct ts_sample_mt TouchScreenSamples[SAMPLES][SLOTS];

    struct ts_sample_mt (*pTouchScreenSamples)[SLOTS] = TouchScreenSamples;
    struct ts_sample_mt *ts_samp[SAMPLES];
    for (i = 0; i < SAMPLES; i++)
            ts_samp[i] = pTouchScreenSamples[i];

and call `ts_read_mt()` like so

    ts_read_mt(ts, ts_samp, SLOTS, SAMPLES);


### Symbols in Versions
|Name | Introduced|
| --- | --- |
|`ts_close` | 1.0 |
|`ts_config` | 1.0 |
|`ts_reconfig` | 1.3 |
|`ts_setup` | 1.4 |
|`ts_error_fn` | 1.0 |
|`ts_fd` | 1.0 |
|`ts_load_module` | 1.0 |
|`ts_open` | 1.0 |
|`ts_option` | 1.1 |
|`ts_read` | 1.0 |
|`ts_read_mt` | 1.3 |
|`ts_read_raw` | 1.0 |
|`ts_read_raw_mt` | 1.3 |
|`tslib_parse_vars` | 1.0 |


***

## building tslib

tslib is cross-platform; you should be able to run
`./configure && make` on a large variety of operating systems.
The graphical test programs are not (yet) ported to all platforms though:

#### libts and filter plugins (`module`)

This is the hardware independent core part: _libts and all filter modules_ as
_shared libraries_, build on the following operating systems.

* **GNU / Linux**
* **Android / Linux**
* **FreeBSD**
* **GNU / Hurd**
* **Haiku**
* **Windows**
* Mac OS X (?)

#### input plugins (`module_raw`)

This makes the thing usable in the read world because it accesses your device.
See our configure.ac file for the currently possible configuration for your
platform.

* GNU / Linux - all (most importantly `input`)
* Android / Linux - all (most importantly `input`)
* FreeBSD - almost all (most importantly `input`)
* GNU / Hurd - some
* Haiku - some
* Windows - non yet

Writing your own plugin is quite easy, in case an existing one doesn't fit.

#### test programs and tools

* GNU / Linux - all
* Android / Linux - all (?)
* FreeBSD - all (?)
* GNU / Hurd - ts_print_mt, ts_print, ts_print_raw, ts_finddev
* Haiku - ts_print_mt, ts_print, ts_print_raw, ts_finddev
* Windows - ts_print.exe, ts_print_raw.exe ts_print_mt.exe

help porting missing programs!

#### libts user plugin
This can be _any third party program_, using tslib's API. For Linux, we include
`ts_uinput`, but Qt, X11 or anything else can use tslib's API.
