---
permalink: /noiseCreation/
title: "Noise Creation"
toc: true
toc_label: "contents:"
last_modified_at: 6/1/2021
---

This here concerns mostly the how of making the noise.

# Linux audio

## Places of interest

| place | description |
| ------------------------------------------- | ----------------------------------------------------- |
| [Extended SamplerBox](https://github.com/hansehv/SamplerBox) | turn your raspberry pi into a midi sampler |
| [Bitwig](https://www.bitwig.com/) | fully featured DAW that runs natively on linux and facilitates modular sound design |
| [KXStudio](https://kx.studio/Applications) | free and open source linux audio+MIDI infrastructure software supporting RT-Kernel and Jack |
[PureData](https://puredata.info/) | open source visual programming for multimedia (MaxMSP alternative) |
[Jurassic Panner](https://github.com/j-p-higgins/jurassic_panner) | Multi input and output panner w/ bird flight dynamics |



## Patching the kernel for real time audio on raspberry pi (PREEMPT_RT)

This specific example was done for the Raspberry Pi 4 (RPi4) and build was done locally (build time is somewhat reasonable). It is especially helpful to patch the recent v5 kernel as it supports booting from EEPROM/USB for RPi4.

**Note**:  PREEMPT_RT may soon merge with the mainline 5 kernel, but as of writing, this is still productive for optimizing the kernel for real-time audio (ie. prioritizing audio system calls)

- [Generic RPi kernel building guide](https://www.raspberrypi.org/documentation/linux/kernel/building.md)

- [Generic RPi kernel patching guide](https://www.raspberrypi.org/documentation/linux/kernel/patching.md)

### Overview 

1.  Find corresponding RT-patch and kernel verions
2.  Patch kernel w/ PREEMPT_RT patch
3.  Build patched kernel
4.  Backup new build
5.  Install
6.  Make use of the patch w/ rtirq-init


### Step-by-step

1.  Clone respective branch and checkout respective subversion commit (there may be a more efficient cloning approach, but I'm still learning git):
    ```
    $ git clone --branch rpi-5.4.y https://github.com/raspberrypi/linux
    $ cd linux
    $ git checkout c8ce98b5387a016760f651e02f6bf99b9fbe559a
    ```
2.  Download patch for respective subversion and patch kernel:

    ```
    $ wget https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.4/older/patch-5.4.61-rt37.patch.gz
    $ gunzip patch-5.4.61-rt37.patch.gz
    $ cat patch-5.4.61-rt37.patch | patch -p1
    ```
3.  Define variables respective to hardware running the kernel. Below is for RPi4:
    ```
    $ KERNEL=kernel7l
    $ make bcm2711_defconfig
    $ make menuconfig
    ```
4.  Configure kernel settings to your liking.  For PREEMPT_RT/Real-time the first of the below is required:
    - General setup &rarr; Preemption Model &rarr; Fully Preembtible Kernel (Real-time).  
    - Press esc twice to return to main menu
    - Kernel features &rarr; Timer frequency &rarr; 1000 Hz

5.  Change the version name of this kernel build so that existing kernel and modules will not be overwritten. Edit the `.config` file as such:
    ```
    CONFIG_LOCALVERSION="-v7l-RT"
    ```

6.  Build the patched kernel:
    ```
    #May want to run this w/ gnu screen so that you can disconnect from the shell.
    $ make -j4 zImage modules dtbs
    #Find a book and start reading. Come back after several chapters.
    $ mkdir modules
    #this will install kernel modules within modules folder to prevent any potential overwriting
    $ sudo make INSTALL_MOD_PATH=modules modules_install
    $ cp arch/arm/boot/zImage arch/arm/boot/$(ls modules/lib/modules).img
    ```

7.  Backup the new build with a script like this:
    ```
    $ cat kernelBackup.sh

    #!/bin/bash
    dateformat=$(date +'%Y%m%d')
    patchName=$(ls patch*)
    kernelVers=$(ls modules/lib/modules)

    tar -cvpzf kernelBk_${kernelVers}_${dateformat}.tar.gz \
    $patchName \
    arch/arm/boot/dts \
    modules \
    arch/arm/boot/$kernelVers.img
    ```

    run it:
    ```
    $ chmod +x kernelBackup.sh
    $ ./kernelBackup.sh
    ```

8.  Install patched kernel and respective dependencies:
    ```
    $ sudo cp arch/arm/boot/$(ls modules/lib/modules).img /boot/$(ls modules/lib/modules).img
    $ sudo cp arch/arm/boot/dts/*.dtb /boot/
    $ sudo cp arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
    $ sudo cp arch/arm/boot/dts/overlays/README /boot/overlays/
    $ sudo cp -r modules/lib/modules/* /lib/modules/.
    ```

9.  Make use of the rt-kernel with this:
    ```
    $ sudo apt install rtirq-init
    ```

## MIDI &rarr; NRPN (quick and dirty) 

**Objective**: Use a legacy MIDI controller (that doesn't support NRPN) with your 21st century sound module.  This was written to meet the need of gradually changing FM synthesis modulation frequency offsets (which could solely be changed w/ NRPN messages) on multiple tracks simultaneously using an old midi keyboard that doesn't support NRPN (Radium49).

### Helpful MIDI tools 

| tool | description |
| ------------------------------------------- | ----------------------------------------------------- |
| aconnect | list connected midi devices:  `aconnect -l` and link inputs (`X`) to outputs (`Y`): `aconnect X Y` |
| aseqdump | monitor MIDI input at client `20:0`:  `aseqdump -p 20:0` |
| [Mido](https://mido.readthedocs.io/en/latest/) | Python MIDI library |



1.  Install needed libraries (if you get nothing from `$ which aconnect`, use your package manager to install `alsa-utils`)
    ```
    $ pip install python-rtmidi
    $ pip install mido
    ```

2.  Create a quick python script ([see here](https://www.elektronauts.com/t/nrpn-tutorial-how-to/46669) and [here](https://psrtutorial.com/music/articles/howToNRPN.html) if you want to better fine tune NRPN):

    `$ cat MIDI_to_NRPN.py`:

    ```
    import mido
    mido.set_backend('mido.backends.rtmidi')

    globalchan=0
    CCtoConvert=82
    NRPNparamMSB=1
    NRPNparamLSB=97
    MIDIkeyboardName="Keystation"
    SoundModuleName="Digitone"

    #find respective output
    inputName = [s for s in mido.get_input_names() if MIDIkeyboardName in s]
    outputName = [s for s in mido.get_output_names() if SoundModuleName in s]

    outport = mido.open_output(outputName[0])

    #define midi CC's for an NRPN message:
    nrpn_select_msb_cc = 99 #send first
    nrpn_select_lsb_cc = 98 #send second
    data_entry_msb_cc = 6 #send third (for coarse adjustment)
    data_entry_lsb_cc = 38 #optional for fine adjustment

    def send_nrpn(nrpn_MSB,nrpn_LSB,msb_value,chan=globalchan,lsb_value=None):
        msbmsg=mido.Message('control_change',channel=chan,control=nrpn_select_msb_cc,value=nrpn_MSB)
        lsbmsg=mido.Message('control_change',channel=chan,control=nrpn_select_lsb_cc,value=nrpn_LSB)
        msbValMsg=mido.Message('control_change',channel=chan,control=data_entry_msb_cc,value=msb_value)
        outport.send(msbmsg)
        outport.send(lsbmsg)
        outport.send(msbValMsg)
        if lsb_value is not None:
        lsbValMsg=mido.Message('control_change',channel=chan,control=data_entry_msb_cc,value=lsb_value)
        outport.send(lsbValMsg)

    def filterCCmsg(port):
        for message in port:
            if message.type is 'control_change' and message.control is CCtoConvert:
                yield message

    #test _ works with:
    #send_nrpn(1,97,127,2,3)
    #send_nrpn(1,97,127,2,55)
    #outport.close()

    try:
        with mido.open_input(inputName[0]) as port:
            print('Waiting for control...')

            for message in filterCCmsg(port):
                print('Received: ' + str(message.value) + ' | On channel: ' + str(message.channel))
                send_nrpn(NRPNparamMSB,NRPNparamLSB,message.value,message.channel)
    except KeyboardInterrupt:
        outport.close()
        pass
    ```

4.  Plug-in your controller and sound module to the RPi MIDI host

5.  Connect the devices virtually on the MIDI host:
    
    List devices:
    ```
    aconnect -l
    ```

    Connect input controller to output sound module (in my case controller is 20 and module is 24):
    ```
    aconnect 20 24
    ```

7.  Configure sliders/knobs on controller for desired MIDI channel and CC# (in my case channels 0-3 and 82) 

6.  Run the script: `python MIDI_to_NRPN.py`, move those knobs, make the bleep boops. 






