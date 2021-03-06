# Gen2 UHF RFID Reader
This is a Gen2 UHF RFID Reader. It is able to identify commercial Gen2 RFID Tags with FM0 line coding and 40kHz data rate (BLF), and extract their EPC. It requires USRPN200 and a RFX900 or SBX daughterboard or a bladeRF.

This project is based on the https://github.com/nkargas/Gen2-UHF-RFID-Reader fork of the original RFID Gen2 Reader available at https://github.com/ransford/gen2_rfid. The reader borrows elements from the software developed by Buettner, i.e. Data flow: Gate -> Decoder -> Reader as well as the conception regarding the detection of the reader commands. CRC calculation and checking functions were also adapted from https://www.cgran.org/browser/projects/gen2_rfid/.

The main purpose of this fork (https://github.com/AdamLaurie/Gen2-UHF-RFID-Reader) is to implement the reader on lower cost hardware such as bladeRF and to pass on (painful) lessons learned in doing so!

The main lesson being that timing is everything...

From looking at multiple versions and derivations of this project it became clear that many people had followed the same path as I had, and had managed to get the RF portion of the project "working", but could not get the overall reader to function. This was clear from the fact that they were attempting to replace the filters with hi/low/band-pass etc., and, presumably, finding the same nice clean RN16 signals they were already getting from the original code but mysteriously the reader failed to ellicit an EPC code from the tag.

After banging my head against that particular brick wall for a couple of weeks, I went right back to the original project, and there it was, in green and black (I'm an old school glass-tty boy):

	Reducing Latency:

	 Reliably meeting the timing requirements of the Gen 2 protocol is
	 difficult using the standard USRP/GNURadio configuration. This is
	 because the default behavior is to look at the signal processing
	 graph, determine the maximum amount of samples that it can consumed,
	 and then block on the USRP until that maximum is received before
	 scheduling the flowgraph. To meet the timing requirements you need to 
	 modify the scheduler behavior so that at every call to the USB subsystem 
	 whatever samples are available are immediately passed to the application. 
	 I have included files that make this modification: 
	 
	 ./rfid/misc_files/usrp_source_base.cc needs to be copied to gnuradio/gr-usrp/src/

	 ./rfid/misc_files/fusb_linux.cc needs to be copied to gnuradio/usrp/host/lib/

	 Once you rebuild gnuradio, this should result in gen2_reader.py having latency
	 ~200-300 us. Check out the files to see how it works. Note that you will be 
	 overwriting the standard files of GNU Radio. In general, the modified files should
	 work as well as or better than the standard mechanism. 

So unless you'd patched your gnuradio installation to do special USB handling for the USRP your reader was never (or possibly only very rarely) going to work. I was seeing a similar problem with the bladeRF.

Happily, with the bladeRF, we have some simple parameters to control the buffering so we can fix this with a command line argument instead of having to patch code, so we simply do this when iterating our source/sink:

```
  self.source = osmosdr.source( args="numchan=" + str(1) + " " + 'bladerf=0,buffers=2,buflen=1024' )
  self.sink = osmosdr.sink( args="numchan=" + str(1) + " " + 'bladerf=0,buffers=2,buflen=1024' )
```

The default is 32 buffers of 4K and we've reduced them to 2 buffers of 1K which is the minimum and seems to be enough to do the trick.
  
The other major issue I had was with gain settings. It became clear that libbladerf1.0 was doing weird things with gain and a lot of time was wasted trying to repeat tests that had worked earlier but were now not performing the in the same way. In short, some of the gain stages appeared to be randomly set based on what had gone before, so repeatability became an issue. Switching to libbladerf2.0 made a huge difference as a single setting of "overall gain" gets automatically allocated to the lower stages, so gain could now be reliably and repeatably set (although a setting of '0' seems to be ignored and the current value won't change).

Finally, it's important to maintain your DC calibration on the bladeRF. To do this, simply run the reader script once to set the frequency and gains, then go into the bladerf-cli and calibrate:

```
$ bladeRF-cli -i
bladeRF> cal lms

  LPF tuning module: 25

  TX LPF I filter: 35
  TX LPF Q filter: 37

  RX LPF I filter: 33
  RX LPF Q filter: 33

  RX VGA2 DC reference module: 27
  RX VGA2 stage 1, I channel: 19
  RX VGA2 stage 1, Q channel: 33
  RX VGA2 stage 2, I channel: 33
  RX VGA2 stage 2, Q channel: 33

bladeRF> cal dc rx

RX DC I: Value =   864, Error =  0.101
RX DC Q: Value =  2016, Error =  8.987

bladeRF> cal dc tx

TX DC I: Value =    32, Error =  0.454
TX DC Q: Value =   800, Error =  0.458

bladeRF> q
```

do this every time you change the frequency or RX/TX gain or power cycle the bladeRF.

I also used a circulator to protect the RX stage of the bladeRF as I was using maximum gain settings for TX:

https://www.pasternack.com/circulator-19db-isolation-960-mhz-10-watts-sma-female-pe83cr007-p.aspx

The modified reader script is in apps/reader-bladerf.py

### Implemented GNU Radio Blocks:

- Gate : Responsible for reader command detection.  
- Tag decoder : Responsible for frame synchronization, channel estimation, symbol period estimation and detection.  
- Reader : Create/send reader commands.

## Installation

- install log4cpp (http://log4cpp.sourceforge.net/)
- install gnuradio from source - this was tested with gnuradio-3.7.13.4
    you may need to install boost from source as well, depending on your distro
    (I was on Ubuntu 16.04.6 LTS) and built boost version 1.69.0
- install libbladerf from source - ensure you end up with libbladeRF.so.2 and NOT libbladeRF.so.1
- alternatively (tested with Ubuntu 18.04) use PPA:
```
sudo add-apt-repository -y ppa:bladerf/bladerf
sudo add-apt-repository -y ppa:myriadrf/drivers
sudo add-apt-repository -y ppa:myriadrf/gnuradio
sudo apt-get update
sudo apt-get remove libbladerf
sudo apt-get install libbladerf2 gr-osmosdr
```
Clone this repo then:
```
cd Gen2-UHF-RFID-Reader/gr-rfid/
mkdir build
cd build/
cmake ../ (logging should be enabled)
sudo make install
sudo ldconfig
```

## Configuration (apps/reader.py or apps/reader-bladerf.py for bladeRF)

- Set USRPN200 address
- Set frequency
- Set tx amplitude
- Set tx gain
- Set rx gain
- Set SELECT funtionality to True/False and create mask as appropriate
- Set maximum number of queries in include/global_vars.h
- Set number of inventory round slots in include/global_vars.h

## How to run

- Real time execution:  

If you use an SBX daughterboard uncomment  #self.source.set_auto_dc_offset(False) in reader.py file

```
cd Gen2-UHF-RFID-Reader/gr-rfid/apps/

sudo GR_SCHEDULER=STS nice -n -20 python ./reader-bladerf.py
```

Note: if you get a core dump try running without 'GR_SCHEDULER=STS'. This is probably not ideal but seems to work!

Hitting \<ENTER> will force the reader back into 'START' mode, or 'Q' will quit. EPC codes will be printed out in realtime
as they are read, and a failed decode is signified by a '!' being printed (these are normally caused by non-response from
TAG rather than an incorrectly decoded response as can be observed by plotting the 'source' output using the scripts in
misc/code).

After termination, part of EPC message (hex value of EPC[104:111]) of identified Tags is printed.  

A successful run should look something like:

```
$ sudo nice -n -20 python ./reader-bladerf.py 
gr-osmosdr v0.1.4-127-g4d83c606 (0.1.5git) gnuradio 3.7.13.4
built-in source types: file fcd rtl rtl_tcp uhd hackrf bladerf rfspace airspy redpitaya 
[bladeRF common] init: DEBUG: entering initialization
[bladeRF source] init: Opening Nuand bladeRF with device identifier string '*:instance=0'
[bladeRF source] Device: Nuand bladeRF Serial # f342...2f1a FW v2.3.1 FPGA v0.10.2
[bladeRF source] init: Clamping "transfers" to 1. Try using a smaller "transfers" value if timeouts occur.
[bladeRF source] init: Buffers: 2, samples per buffer: 1024, active transfers: 1
[bladeRF source] bladerf_source_c: DEBUG: initialization complete
gr-osmosdr v0.1.4-127-g4d83c606 (0.1.5git) gnuradio 3.7.13.4
built-in sink types: uhd hackrf bladerf redpitaya file 
[bladeRF common] init: DEBUG: entering initialization
[bladeRF sink] init: Opening Nuand bladeRF with device identifier string '*:instance=0'
[bladeRF sink] Device: Nuand bladeRF Serial # f342...2f1a FW v2.3.1 FPGA v0.10.2
[bladeRF sink] init: Clamping "transfers" to 1. Try using a smaller "transfers" value if timeouts occur.
[bladeRF sink] init: Buffers: 2, samples per buffer: 1024, active transfers: 1
[bladeRF sink] bladerf_sink_c: DEBUG: initialization complete
[INFO @ host/libraries/libbladeRF/src/board/bladerf1/bladerf1.c:1962] Clamping bandwidth to 1500000 Hz
[bladeRF source] start: DEBUG: starting source
[bladeRF sink] start: DEBUG: starting sink
<ENTER> to re-start, 'Q' to quit 
!!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!!!!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +!!!!!+ e2-00-41-45-77-0e-01-61-11-30-a1-01 +
!!!!!!!
 --------------------------
| Number of queries/queryreps sent : 249
| Current Inventory round : 250
 --------------------------
| Correctly decoded EPC : 68
| Number of unique tags : 1
| Tag ID : 1  Num of reads : 68
 --------------------------
<ENTER> to re-start, 'Q' to quit 
```


- Offline:  
    Change DEBUG variable in apps/reader.py to TRUE (A test file already exists named file_source_test).  
    The reader works with offline traces without using a USRP.  
    The output after running the software with test file is:  
    
    | Number of queries/queryreps sent : 71  
    | Current Inventory round : 72  

    | Correctly decoded EPC : 70  
    | Number of unique tags : 1  
    | Tag ID : 27  Num of reads : 70  
 
## Logging

- Configuration file : /home/username/.gnuradio/config.conf  
    Edit the above file and add the following lines  

    [LOG]  
    debug_file = /PathToLogFile/Filename  
    debug_level = info  
    
    Logging may cause latency issues if it is enabled during real time execution!

## Debugging  

The reader may fail to decode a tag response for the following reasons

1) Latency: For real time execution you should disable the output on the terminal. If you see debug messages, you should either install log4cpp or comment the corresponding lines in the source code e.g., GR_LOG_INFO(d_debug_logger, "EPC FAIL TO DECODE");

2) Antenna placement. Place the antennae side by side with a distance of 50-100cm between them and the tag 2m (it can detect a tag up to 6m) away facing the antennae.

3) Parameter tuning. The most important is self.ampl which controls the power of the transmitted signal (takes values between 0 and 1).

If the reader still fails to decode tag responses, uncomment the following line in reader.py file

 #self.connect(self.source, self.file_sink_source)

Run the software for a few seconds (~5s). A file will be created in misc/data directory named source. This file contains the received samples. You can plot the amplitude of the received samples using the script located in misc/code folder. The figure should be similar to the .eps figure included in the folder. Plotting the figure can give some indication regarding the problem. You can also plot the output of any block by uncommenting the corresponding line in the reader.py file. Output files will be created in misc/data folder:

- /misc/data/source  
- /misc/data/matched_filter  
- /misc/data/gate 
- /misc/data/decoder  
- /misc/data/reader
    
A good read should look like this (plot generated with Octave from 'misc/data/source'):

![UHF Backscatter](uhf-annotated.png)

or this if you have SELECT enabled (see comments in reader-bladerf.py):

![UHF Backscatter with SELECT](uhf-select-annotated.png)

If you are missing the EPC traces the most likely cause is a timing issue within the 'CRITICAL TIMING' section, as described above. If you are missing the RN16 traces your TAG is probably too far away and/or your TX gain is not high enough.
    
## Hardware:

  - 1x USRPN200/N210  
  - 1x RFX900/SBX daughterboard  

or

  - 1 x bladeRF x115

and

  - 2x circular polarized antennae

or

  - 1x circulator (see note above)
  - 1x circular polarized antenna

## Tested on:
  Ubuntu 16.04 64-bit  
  GNU Radio 3.7.13.4
  Boost 1.69.0
  bladeRF 2018.12-rc3-15-g9dd4372 (git)
  
## If you use this software please cite:
N. Kargas, F. Mavromatis and A. Bletsas, "Fully-Coherent Reader with Commodity SDR for Gen2 FM0 and Computational RFID", IEEE Wireless Communications Letters (WCL), Vol. 4, No. 6, pp. 617-620, Dec. 2015. 

## Contact:
  Adam Laurie (email: adam@algroup.co.uk)
