# KiwiSDR SNR Monitoring Utilities
Based on work by Olaf - LA3RK

## Install Dependencies
This gets it going on a Raspbian system...

* `sudo apt-get install python3-rrdtool rrdtool python3-numpy python3-matplotlib`
* `sudo pip3 install suntime`


## Gathering data
```
$ python3 snrtorrd.py -s your.kiwisdr.hostname -p 8073
```

The default frequency span is 0 - 30 MHz. The number of FFT bins is fixed at 1024.

Optional arguments include:
```
Options:
  -h, --help            show this help message and exit
  -s SERVER, --server=SERVER
                        server name
  -p PORT, --port=PORT  port number
  -a PASSWORD, --password=PASSWORD
                        server password
  -l LENGTH, --length=LENGTH
                        how many samples to draw from the server (Default: 100)
  -t STEP, --timestep=STEP
                        Expected timestep between samples
  --timeout=TIMEOUT     Connection Timeout
  -z ZOOM, --zoom=ZOOM  zoom factor
  -o START, --offset=START
                        start frequency in kHz
  -v VERBOSITY, --verbose=VERBOSITY
                        whether to print progress and debug info
  --spectra=SPECTRA     Spectra Output File (csv)
```

For example to login to a private server, and save out raw spectra data, you would use:
```
$ python3 snrtorrd.py -s your.kiwisdr.hostname -p 8073 -a yourpassword --spectra myserver_spectra.csv
```


## RRD Plotting
```
$ python3 rrdtograph.py -s your.kiwisdr.hostname --title "My KiwiSDR SNR"
```
Note that the hostname must match the hostname used above, as this script uses filenames based on the hostname.
This script will generate plots with filenames of the form:
```
your.kiwisdr.hostname_0_30000-Daily.png
your.kiwisdr.hostname_0_30000-Weekly.png
your.kiwisdr.hostname_0_30000-Monthly.png
```

## Spectrograph Plotting
If you have been saving spectra data with the `--spectra` option above, you can plot out a neat spectrograph showing HF band activity over the last few days. This utility can also plot an estimate of total RX power into the KiwiSDR

```
$ python3 kiwi_spectrum_plot.py --hours 72 --spectrograph my_spectro.png --title "My KiwiSDR"  --rxpower my_rxpower.png myserver_spectra.csv
```

Arguments are:
```
positional arguments:
  filename              KiwiSDR Spectrum Log File (CSV)

optional arguments:
  -h, --help            show this help message and exit
  --hours HOURS         How many hours to plot. (default: 72)
  --cmap_min CMAP_MIN   Colormap Minimum Value (dBm) (default: -110)
  --cmap_max CMAP_MAX   Colormap Maximum Value (dBm) (default: -30)
  --clip                Clip file to the hour limit specified with --hours
                        (default: False)
  --spectrograph SPECTROGRAPH
                        Save Spectrograph to this file. (default: None)
  --rxpower RXPOWER     Save RX Power Plot to this file. (default: None)
  --title TITLE         KiwiSDR Name (for plot titles) (default: KiwiSDR)
  -v, --verbose         Verbose output (set logging level to DEBUG) (default:
                        False)
```

To avoid the spectra CSV file getting really big, you can use the `--clip` option, which will remove any entries in the spectra file older than the specified number of hours.

Note that the Spectrograph plot is optimized for 72 hours of span with 10 minute time steps, and will look a bit odd until this much data has been gathered.

## Putting it all together
You will probably be wanting to run the above scripts automatically. This can be done using a cronjob.

For example, for a single KiwiSDR, the following could be put into a bash script (e.g. `run_snr.sh`):
```
#!/usr/bin/env bash

HOSTNAME=my.kiwisdr.server.org
PORT=8073

cd /home/user/kiwisnr-rrd/

# Capture Data
python3 snrtorrd.py -s $HOSTNAME -p $PORT --spectra=kiwisdr_spectra.csv
# Produce RRD Plots
python3 rrdtograph.py -s $HOSTNAME --title "My KiwiSDR"
# Produce Spectrograph and RX Power plots
python3 kiwi_spectrum_plot.py --hours 72 --spectrograph my_spectro.png --title "My KiwiSDR"  --rxpower my_rxpower.png kiwisdr_spectra.csv

# Copy plots to a remote server for display
scp *.png myserver:~/some/path/
```

This script could then be run in a cronjob with an entry like:
```
# m h  dom mon dow   command
*/10 * * * * /home/user/run_snr.sh
```

## Docker

```sh
docker build -t kiwisdr-rrd:latest .
docker run --rm -v $PWD/output:/output -v $PWD/data:/data  kiwisdr-rrd:latest host.name 8073 "pretty name" password
```