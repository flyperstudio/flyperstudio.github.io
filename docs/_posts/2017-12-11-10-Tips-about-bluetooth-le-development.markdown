---
layout: post
title:  "10 Tips about bluetooth le development"
date:   2017-12-11 15:25:16 +1100
categories: tech blog
---
Currently, I was helping my client to develop an SDK to communicate with Bluetooth le device. I developer three SDKs, one is Xamarin, iOS native with Objective-c and Java SDK for Android. Here are some tips I found when I was developing these SDKs. I hope these will save the time for firmware and mobile developers.

## 1. Put Mac address into advertising manufacturer specific data.

As there is no way to get Mac address from iOS SDK (until iOS 11). In iOS, we can use [CBPeripheral.identifier](https://developer.apple.com/documentation/corebluetooth/cbpeer/1620687-identifier) to tell the difference between devices, but it cannot cross the platforms. If you want to do some web API works and need to identify the device, the Mac address is the best choice. So the simple solution is put the Mac address into advertising.

## 2. Advertising manufacturer specific data cannot be longer than 26 bytes.
From the document, the advertising packer can be up to 31 bytes. But from our testings some Android devices, like Motorola Moto G4, it cannot handle that length. So 25 is the maximum length of advertising manufacturer specific data.

## 3. Put some random bytes into advertising data when you want to test background task in iOS.
iOS has strict restrictions on Bluetooth background task. Our use case is simple, like when a user approaching some device, the App can raise a notification. But it cannot test successfully every time. The best solution for this use case is making the device follow iBeacon design, but we don't want to do that.

Another solution is when App found a device, then keep the connection of that, if the user leaves and back it will auto-connect the device, we don't want to do this either, cause it will run lots of battery.

Back to the topic, if you want to do long-term iOS Bluetooth background task, you do better follow the "Performing Long-Term Actions in the Background" section in this [document](https://developer.apple.com/library/content/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html).

But why we need to put some random bytes into advertising data, because this API, [scanForPeripheralsWithServices:options:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/1518986-scanforperipheralswithservices?language=objc), the [CBCentralManagerScanOptionAllowDuplicatesKey](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerscanoptionallowduplicateskey) option will set to NO in the background task. So it means if your advertising data are all the same each time, it won't invoke [centralManager:didDiscoverPeripheral:advertisementData:RSSI:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerdelegate/1518937-centralmanager?language=objc)  delegate. But even we do this, the background task still cannot work as we want, I have created a TSI to Apple, but the answer is this is by design to save the battery, and here is the [condition](https://developer.apple.com/library/content/qa/qa1962/_index.html) can relaunch the App from the Bluetooth background task.

Another thing is, for advertising interval, you can pick one of them can work well both platforms.
* 152.5 ms
* 211.25 ms
* 318.75 ms
* 417.5 ms
* 546.25 ms
* 760 ms
* 852.5 ms
* 1022.5 ms
* 1285 ms

They come from [Apple Bluetooth design guidelines](https://developer.apple.com/hardwaredrivers/BluetoothDesignGuidelines.pdf).

## 4. Manufacturer data are different from Android and iOS.
I put analysis manufacturer data into PCL, but I found Android and iOS are different,

Android after Lollipop (API 21), it put manufacturer data into a key-value array, you should combine them to get the manufacturer data with iOS, here is the C# code

```csharp
byte[] advBytes = new byte[manufaturerData.ValueAt(0).ToArray<byte>().Length + 2]; 
byte[] begin = BitConverter.GetBytes((ushort)manufaturerData.KeyAt(0)).Take(2).ToArray(); 
Array.Copy(begin, advBytes, 2); 
Array.Copy(manufaturerData.ValueAt(0).ToArray<byte>(), 
          0, 
          advBytes, 
          2, 
          manufaturerData.ValueAt(0).ToArray<byte>().Length);
```

## 5. Call different API in Android under Lollipop.
Android support Bluetooth le since Android 4.3 (API 18), so before Lollipop(API 21), you should call StartLeScan instead of StartScan.
Another thing is
boolean startLeScan (UUID[] serviceUuids, BluetoothAdapter.LeScanCallback callback)
The serviceUuids filter is not working, so we need to do the filter by ourselves.

Here is the code to get the services Uuid from adv data in C#

```csharp
        // Solution from https://stackoverflow.com/questions/18019161/startlescan-with-128-bit-uuids-doesnt-work-on-native-android-ble-implementation
        List<string> ParseUUIDs(byte[] advertisedData)
        {
            List<string> uuids = new List<string>();
            int offset = 0;
            while (offset < (advertisedData.Length - 2))
            {
                int len = advertisedData[offset++];
                if (len == 0)
                    break;

                int type = advertisedData[offset++];
                switch (type)
                {
                    case 0x02: // Partial list of 16-bit UUIDs
                    case 0x03: // Complete list of 16-bit UUIDs
                        while (len > 1)
                        {
                            int uuid16 = advertisedData[offset++];
                            uuid16 += (advertisedData[offset++] << 8);
                            len -= 2;
                            uuids.Add($"0000{uuid16.ToString("X")}-0000-1000-8000-00805f9b34fb");
                        }
                        break;
                    case 0x06:// Partial list of 128-bit UUIDs
                    case 0x07:// Complete list of 128-bit UUIDs
                              // Loop through the advertised 128-bit UUID's.
                        while (len >= 16)
                        {
                            try
                            {
                                // Wrap the advertised bits and order them.
                                Java.Nio.ByteBuffer buffer = Java.Nio.ByteBuffer.Wrap(advertisedData,
                                                                                      offset++, 16).Order(Java.Nio.ByteOrder.LittleEndian);
                                long mostSignificantBit = buffer.Long;
                                long leastSignificantBit = buffer.Long;
                                uuids.Add(new UUID(leastSignificantBit,
                                        mostSignificantBit).ToString());
                            }
                            catch (Java.Lang.IndexOutOfBoundsException e)
                            {
                                continue;
                            }
                            finally
                            {
                                // Move the offset to read the next uuid.
                                offset += 15;
                                len -= 16;
                            }
                        }
                        break;
                    default:
                        offset += (len - 1);
                        break;
                }
            }

            return uuids;
        }
```

## 6. Android system has the huge problem with connection.
At the beginning of the project, we have 1 service and 1 characteristic, it seems to work fine on both platforms. As we put more features in, we put more characteristics in the single service, the Android SDK cannot connect with devices successfully everytime. It's not my code's problem, I used some third-party tools like LightBlue, Ble Scanner, nRF Connect, they do the same thing.

The SDK can connect the devices after I call ConnectGatt API, but when I call DiscoverServices API, it disconnects the devices from low-level code.

I have tried some solutions for this, like
* RequestConnectionPriority after connect
* Delay 500ms to call DiscoverServices API

They make connection stable a little bit but still failed many times.

We are trying to reduce the characteristics to see what happens.

Why not use single characteristic? As we have many features in the devices. Multiple characteristics will be easy for testing with the third-party tool. You can name every characteristic. If using single characteristic, it will introduce more logic for both firmware side and mobile SDK side, and even make testing harder.

## 7. Auto disconnect from device
It's a good idea to disconnect from App after few minutes, cause it can save the battery. The device can implement this logic, for the App need to handle the disconnect event. But in Android, if you set auto-connect as true in [ConnectGatt API.](https://developer.android.com/reference/android/bluetooth/BluetoothDevice.html#connectGatt(android.content.Context,%20boolean,%20android.bluetooth.BluetoothGattCallback))

It will auto-connect the device, even the device drops the connection by itself. So the solution is you can call `Disconnect()` function in OnConnectionStateChange delegate like this

```csharp
gatt.Disconnect(); 
gatt.Dispose(); 
tracker.Gatt = null;
```

## 8. Notification of characteristics is different between iOS and Android.
In iOS we cannot tell notification and read, cause they are coming from the same delegate.

Enable characteristics notification is sample in iOS, just call SetNotifyValue API is OK. 

`peripheral.SetNotifyValue(enable, cbc);` 

But in Android, you need to write the descriptor. Here is the C# code

```csharp
bool EnableNotification (Tracker tracker, BluetoothGattCharacteristic cbc, bool enable) {
    if (!cbc.Properties.HasFlag (GattProperty.Notify))
        return true;
    var descriptor = cbc.GetDescriptor (UUID.FromString ("00002902-0000-1000-8000-00805f9b34fb"));
    if (enable &&
        descriptor.GetValue () != null &&
        descriptor.GetValue () [0] == BluetoothGattDescriptor.EnableNotificationValue.ToArray () [0])
        return true;
    if (!enable &&
        descriptor.GetValue () != null &&
        descriptor.GetValue () [0] == BluetoothGattDescriptor.DisableNotificationValue.ToArray () [0])
        return true;
    bool res = tracker.Gatt.SetCharacteristicNotification (cbc, enable);
    if (enable) {
        res & = descriptor.SetValue (BluetoothGattDescriptor.EnableNotificationValue.ToArray ());
    } else {
        res & = descriptor.SetValue (BluetoothGattDescriptor.DisableNotificationValue.ToArray ());
    }
    WriteDescriptEvent = new AutoResetEvent (false);
    res & = tracker.Gatt.WriteDescriptor (descriptor);
    WriteDescriptEvent.WaitOne ();
    return res;
}
```

## 9. Read and write operation on Android is not thread-safe.
In Android, you should put single read or write operation into a single thread. For example, you call ReadCharacteristic API, then you get the result from the callback, currently the callback and read API are in the same thread. If you call another read API in this callback, it won't return success result. From the source code, there is a lock to make sure the front callback finished and then do another read. So you need to create another thread and do another read.

The lock applies to Read, write, read/write Descriptor APIs. Sometimes, even you create a new thread, it still not working, you need to delay few seconds to do another operation.

## 10. Make sure write with response must have the response.
There are two types write operations, one is write without response, another is write with response. In iOS even you tell the SDK wrong write type, it still can work, but in Android, it cannot. It will block all coming operations until timeout. So make sure both firmware and App are the same type for write operation.

## Conclusion

As a developer, I prefer iOS platform. The iOS SDK easy to use and stable, the document is good, even in iOS you cannot do what you want, like the background thing, but at least every issue has its answer.

But in Android, it's a chaos. The document doesn't update like [this](https://developer.android.com/guide/topics/connectivity/bluetooth-le.html). It shows the API before Lollipop. And you cannot find the answer to a simple question, like why it cannot connect successfully every time. And the crazy thing is if a user reports you a bug, you need to buy the same device, but may not reproduce that.

After discovering, connecting, read/writing, the code can focus on operating bytes. If you want to do something with Bluetooth in Xamarin, I recommend [Plugin.BluetoothLE](https://github.com/aritchie/bluetoothle) which use Rx.net. Rx.net is very suitable for read and write operations, cause the result come just like a stream.