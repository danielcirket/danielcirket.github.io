---
layout: post
title: Nativescript - Android, Bluetooth Barcode Scanners and Typescript 
---

In this article I'm going to be using a bluetooth scanner and combined with a background worker to handle reading barcode information into a nativescript application on Android.

**_Note: for this article I'm using a Opticon OPN-2002 bluetooth barcode scanner [see here](http://www.opticonusa.com/products/opticon-legacy-products/OPN-2002.html)._**

_Disclaimer: I will assume that this process should be the same for various bluetooth barcode scanners, however I cannot make any guarantee on this_

A quick summary of what we will achieve in this article:

1. Setup a service to contain and interact with the devices bluetooth
2. Create a background worker which handles listening and messages from the bluetooth device
3. Create a background worker service to abstract away some of the additional setup for background workers
4. Put it all together and listen for input from the barcode scanner

To keep the bluetooth logic consilodated we will firstly create a bluetooth service which will allow us to do the following:

```ts
export interface IBluetoothService {
    IsEnabled : Promise<boolean>; // Getter
    BondedDevices : Array<BondedDevice>; // Getter
    Notify(device : BondedDevice, callback : (data : any) => void) : void;
}
```

The service looks like this:

```ts
import * as app from 'application';
import * as Utils from 'utils/utils';
import { BackgroundWorker } from '../backgroundworker/backgroundworker.service';

export class BluetoothService implements IBluetoothService {

    private readonly _adapter : android.bluetooth.BluetoothAdapter;
    private readonly _manager : any;
    
    public get IsEnabled() : Promise<boolean> {
        return new Promise<boolean>((resolve, reject) => {
            try {
                console.log('IsEnabled: ', this._adapter.isEnabled());
                resolve(this._adapter != null && this._adapter.isEnabled());
            } catch (ex) {
                reject(ex);
            }
        });
    }
    public get BondedDevices() : Array<BondedDevice> {

        const devices = this._adapter.getBondedDevices().toArray();
        const results = new Array<BondedDevice>();
        const length = devices.length;

        for (let i = 0; i < length; i++) {

            const device = <android.bluetooth.BluetoothDevice>(devices[i]);
            
            // If you need the supported features, in this example we don't.
            //const supportedFeatures = device.getUuids();
            //for (let j = 0; j < supportedFeatures.length; j++) {
            //    const supportedFeature = supportedFeatures[j];
            //}

            const address = device.getAddress();
            const name = device.getName();

            results.push(new BondedDevice(address, name));
        }

        return results;
    }

    public Notify(device : BondedDevice, callback : (data : any) => void) : void {
        const worker = new BackgroundWorker(
            '~/workers/bluetooth/bluetooth.worker.js', 
            JSON.stringify(device), 
            (message : any) => {
                callback(message);
            },
            (error : any) => {
                console.log('Error received from worker: ', JSON.stringify(error, null, 4));
            });
    }

    constructor() {
        this._manager = Utils.ad.getApplicationContext().getSystemService((<any>(android.content.Context)).BLUETOOTH_SERVICE);
        this._adapter = this._manager.getAdapter();
    }
}

export class BondedDevice {
    
    public Id : string;
    public Name : string;

    constructor(id : string, name : string) {
        this.Id = id;
        this.Name = name;
    }
}
```

This service allows us to query if bluetooth is enabled, list the bonded devices, and provide a callback for device 'messages'.

Now we have the bluetooth service implemented you may have noticed that we have a reference to the BackgroundWorker, this is a wrapper to allow easier use of background workers - you don't need to use this, but I found it easier to get it right once than keep replicating the setup code for a background worker myself.

The background worker wrapper is here:

```ts
import { Observable } from 'data/observable';

export class BackgroundWorker extends Observable {
    private readonly _workerPath : string;
    private readonly _worker : Worker;
    private readonly _onError : (error : any) => void;
    private readonly _onMessage : (message : any) => void;
    
    private OnMessage(message : MessageEvent) : void {
        this._onMessage(message.data);
    }
    private OnError(error : any) : void {
        this._onError(error);
        console.log('Error in background worker: "' + this._workerPath + '", Error: ', JSON.stringify(error));
        this._worker.terminate();
    }

    /** @description Creates and starts a new background worker.   
     * @param {string} path to worker file
     * @param {data} data to be sent to the worker
     * @param {callback} callback to fire if the worker posts a message back to the main thread
     * @param {callback} callback to fire if the worker raises an error, the worker will be automatically terminated afterwards 
     * @return {BackgroundWorker}  
     */  
    constructor(workerPath : string, data : any, onmessage : (message : any) => void = (msg) => {}, onerror : (error : any) => void = (error) => {}) {
        super();

        if (workerPath == null || workerPath.length < 1) {
            throw new Error('Background worker path cannot be null or empty');
        }

        console.log('Starting worker...');
        this._workerPath = workerPath;
        this._worker = new Worker(this._workerPath);

        console.log('Worker: ', workerPath);
        // Setup event handlers
        this._onMessage = onmessage;
        this._onError = onerror;
        this._worker.onmessage = this.OnMessage.bind(this);
        this._worker.onerror = this.OnError.bind(this);

        console.log('Posting data to worker');

        // Post the data
        this._worker.postMessage(data);

        console.log('Data posted to worker');
    }
}
```

Now that we have both the bluetooth service and a nice wrapper around the web worker we need to setup the bluetooth background worker script itself, this will handle the details of actually interacting with the device, such as connecting and reading input.

The worker script:

```ts
var globals = require("globals");
var Utils = require('utils/utils');
var BondedDevice = require('../../services/bluetooth/bluetooth');

// NOTE(Dan): This is a fix for the type mismatch in the typescript lib definition vs nativescript
declare function postMessage(message: any);

onmessage = function(data : any) : void {
    console.log('Message received in bluetooth worker: ', JSON.stringify(data));

    const manager = Utils.ad.getApplicationContext().getSystemService((<any>(android.content.Context)).BLUETOOTH_SERVICE);   
    const adapter = manager.getAdapter();
    const device = JSON.parse(data.data);
    const remoteDevice = adapter.getRemoteDevice(device.Id);

    try {
        const socket = remoteDevice.createInsecureRfcommSocketToServiceRecord(java.util.UUID.fromString('00001101-0000-1000-8000-00805F9B34FB'));

        socket.connect();

        const inputStream = socket.getInputStream();

        while(true)
        {                   
            if (inputStream.available() > 0) {
                const result = [];

                while(inputStream.available() > 0) {
                    result.push(inputStream.read());
                }
                
                const formattedResult = result.map<string>(value => {
                    return String.fromCharCode(value);
                }).join('');

                postMessage(formattedResult);
            }
        }
    }
    catch (ex) {
        console.log('Exception opening connection: ', ex);
    }
}

onerror = function(error : any) : void {
    console.log('Error in worker: ', JSON.stringify(error));
}
``` 

Now that we have the main building blocks we can put them together in order to read the barcode scanner input.

Depending on how you need to handle reading input, either on one specific screen within you app or handle the input regardless of which screen will determine where and how you setup the interaction between the parts we have put together so far. In this example I'm going to be listening to the input no matter where I am in the application. This will be handled in the app.ts file (My entry point).

```ts
import * as app from 'application';
import { NavigationService } from './services/navigator/navigator.service';
import { BluetoothService, BondedDevice } from './services/bluetooth/bluetooth';

app.on(app.launchEvent, (args : app.LaunchEventData) => {

    const bluetoothService=  new BluetoothService();

    bluetoothService.Enabled
        .then((enabled : boolean) => {
            
            // If bluetooth is disabled, we don't need to do anything
            if (!enabled) {
                return;
            }

            const pairedDevices = bluetoothService.BondedDevices;

            // If we have no paired devices, we don't need to do anything
            if (pairedDevices.length < 1) {
                return;
            }

            let selectedDevice : BondedDevice = null;

            if (pairedDevices.length > 0) {
                selectedDevice = pairedDevices[0];
            }

            if (selectedDevice == null) {
               return;
            }

            bluetoothService.Notify(selectedDevice : BondedDevice, (result : string) => {
                console.log('Result from bluetooth scanner: ', JSON.stringify*(result));

                // TODO: Handle the scanned input as required
            });
        });

});

app.start({ moduleName: './components/login/login', bindingContext: NavigationService.Instance.LoginViewModel });

```

Now we should have a working example, for which when opening the application while you have your bluetooth barcode scanner paired should create a socket connection to the device in a background worker and continue running waiting for scanned input.

There are a number of things that you might want to add which could include reconnecting the bluetooth device if the connection drops, selecting a different device, dealing with pairing in app, prompts for bluetooth permissions and so on.