# Setting up Minicom for accessing DPU

Minicom provides the user to access USB devices, including the DPU, with the correct config.

## Quick start

1. Install dependencies if necessary
`sudo apt-get install minicom`
2. Locate your happy DPU devices
    1. Psyically, go the the testbed and make sure you have connect the DPU to the host with a USB cable
    2. Check the device interface for the DPU
        `sudo dmesg | grep tty` or `ls /dev/ttyUSB*`
        Normally, if there are no other device, it should be the **ttyUSB0**
3. Configure Minicom
   1. `sudo minicom -s`
        Change the setting to this one:
        ![](./img/minicom-config.png)
        Basically, you should change the *Serial Device* and the *Hardware Flow Control*, and probably the *Bits rate*
   2. Press enter to leave, and you can save the config with a name (mine go with the *dpu*) or as a default one.
        P.S. If you don't save the config, it would reset EACH TIME you issue minicom.
4. Login Minicom
   1. You can either reuse the previous terminal, but for simplicity, I recommand quit the previous Minicom session first.
   2. Run `sudo minicom dpu` (replace the dpu with the name you got, or ignore it if you make it a default one)
   3. Wait with patience, the login should pop up anytime soon, you just have to wait.
      1. I am not sure if pressing "enter" is necessary, but it won't hurt for trying
5. Leaving minicom env
   Simply use ctrl+A then press Q. You can use ctrl+A then press Z to get more info, while most command start with the ctrl+A stuff.