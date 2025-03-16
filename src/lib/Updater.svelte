<script lang="ts">
    import usbImage from "../assets/usb-gpt.png";
    import { ESPLoader, Transport, ROM } from "esptool-js";
    import CryptoJS from 'crypto-js';
    import {Card, Button} from 'flowbite-svelte'
    import { Heading} from 'flowbite-svelte';
    import { Select, Label, Hr, Spinner,Progressbar } from 'flowbite-svelte';
    import { ButtonGroup, Toast } from 'flowbite-svelte';
    import {LinkOutline, DownloadOutline, LinkBreakOutline, AtomOutline,FireOutline} from 'flowbite-svelte-icons'
    import {blur} from 'svelte/transition'
    import {sineOut} from 'svelte/easing'
    const urlRobotics = '/assets/firmware.bin';
    const urlIOT = '/assets/firmware.bin';

    let serialPort;
    let loader : ESPLoader;
    let chipInfo;
    let isConnected = $state(false);
    let firstime = true;
    let error = $state(null);
    let chip: string = $state(null);
    let macAddress = $state(null);
    let firmware :string = null;
    let selected = 'robotics';
    let uploading = $state(false);  
    let progress = $state(0);

    let selectables = [
      {value : 'robotics', name : 'MoBOT Robotics'},
      {value : 'iot', name : 'MoBOT IOT'}
    ];
    
    async function connectToESP() {
      try {
        error = null;
        chipInfo = null;
  
        serialPort = await navigator.serial.requestPort();
  
        if (!serialPort) {
          throw new Error("Serial port not available.");
        }
  
        const transport = new Transport(serialPort, true);
        
        const flashOptions = {
            transport,
            baudrate: 921600,
            } as LoaderOptions;
        
        loader = new ESPLoader(flashOptions);

        chip = await loader.main();
        isConnected = true;

        macAddress = (await loader.chip.readMac(loader)).toUpperCase();
        firstime = false;
      } catch (err) {
        error = err.message || "Failed to connect to ESP.";
      }
    }

    async function disconnectESP() {
      try {
        if (serialPort) {
          await serialPort.close();
        }
        isConnected = false;
        chip = null;
        macAddress = null;
      } catch (err) {
        error = "Failed to disconnect.";
      }
    }

    async function updateMobot(){
      try{
        uploading = true;
        if (selected === 'robotics'){
          firmware = await fetchBinaryData(urlRobotics)
        }
        else if (selected === 'iot'){
          firmware = await fetchBinaryData(urlIOT)
        }
        else {
          throw new Error ("No board selected")
        }
        const fileArray = [];
        const progressBars = [];

        fileArray.push({ data: firmware, address: Number(0x1000) });

        const flashOptions: FlashOptions = {
          fileArray: fileArray,
          flashSize: "keep",
          eraseAll: false,
          compress: true,
          reportProgress: (fileIndex, written, total) => {
            progress = (written / total) * 100;
            console.log(progress);
          },
          calculateMD5Hash: (image) => CryptoJS.MD5(CryptoJS.enc.Latin1.parse(image)),
        } as FlashOptions;
        await loader.writeFlash(flashOptions);  
        uploading = false;
      }
      catch(err){
        error = err.message;
        uploading = false
      }
  }

    
    async function fetchBinaryData(url) {
      const response = await fetch(url);
      if (!response.ok) {
        throw new Error(`Failed to fetch binary data from ${url}: ${response.statusText}`);
      }
      const buffer = await response.arrayBuffer();
      const byteArray = new Uint8Array(buffer);
      let binaryString = '';
      for (let i = 0; i < byteArray.length; i++) {
        binaryString += String.fromCharCode(byteArray[i]);
      }
      return binaryString;
    }

  </script>

  <Card size = lg img= {usbImage}>

    <Heading tag="h2" customSize="text-4xl font-extrabold ">Update your MoBOT</Heading>

    <Label>
      Select the product you want to update 

      <Select class="mt-2" items={selectables} bind:value={selected} />

    </Label>

    <Hr classHr="my-8 w-64 h-1" icon>
      <AtomOutline class="w-6 h-6 text-gray-700 dark:text-gray-300" />
    </Hr>

    <ButtonGroup id= "control" class ="m-2">
      {#if !isConnected}
        <Button color="green" on:click={connectToESP}> <LinkOutline class="w-5 h-5 ms-0"/> Connect</Button>
      {:else if isConnected}
        <Button color="red" on:click={disconnectESP}> <LinkBreakOutline class="w-5 h-5 ms-0"/> Disconnect</Button>
      {/if}
      
      <Button disabled = {!isConnected} color="yellow" on:click={updateMobot}>
        {#if uploading}<Spinner class="me-3" color="white" size=4 /> {/if}
        <DownloadOutline class="w-5 h-5 ms-0"/>
        Update
      </Button>
    </ButtonGroup>

    {#if uploading}
      <Progressbar
        {progress}
        animate
        precision={2}
        labelInside
        tweenDuration={1500}
        easing={sineOut}
        size="h-7"
        labelInsideClass="bg-yellow-600 text-yellow-100 text-base font-medium text-center p-1 leading-none rounded-full"
        class="m-3"
      />
    {/if}

    <Hr  class ="m-1 "></Hr>

    <div>
      <h3>Device Information</h3>
      {#if chip}
        <div class="info">
          <p><strong>Chip Model:</strong> {chip}</p>  
          <p>MAC Address: {macAddress}</p>
        </div>
      {/if}
    </div>

  </Card>




  <div class="fixed bottom-4 right-4 z-50">
    {#if error}
    <Toast transition={blur} params={{ amount: 10 }} class="mb-4" >
      <FireOutline slot="icon" class="w-6 h-6 text-primary-500 bg-primary-100 dark:bg-primary-800 dark:text-primary-200" />
      <p class="error">{error}</p>
    </Toast>
    {/if}

    {#if isConnected}
    <Toast transition={blur} params={{ amount: 10 }} class="mb-4" color='green'>
      <LinkOutline slot="icon" class="w-6 h-6" color='green'/>
      MoBOT connected.
    </Toast>
    {/if}
    
    {#if !isConnected && !firstime}
    <Toast transition={blur} params={{ amount: 10 }} class="mb-4" >
      <LinkBreakOutline slot="icon" class="w-6 h-6 text-primary-500 bg-primary-100 dark:bg-primary-800 dark:text-primary-200" />
      MoBOT disconnected.
    </Toast>
    {/if}
  </div>

  <style>
    .info {
      margin-top: 1em;
      font-size: 1.1em;
      color: #0000006e;
    }
    .error {
      color: red;
    }
  </style>

  
  

 