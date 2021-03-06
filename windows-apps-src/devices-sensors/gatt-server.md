---
author: msatranjr
title: Bluetooth GATT Server
description: This article provides an overview of Bluetooth Generic Attribute Profile (GATT) Server for Universal Windows Platform (UWP) apps, along with sample code for common use cases. 
ms.author: misatran
ms.date: 02/08/2017
ms.topic: article
ms.prod: windows
ms.technology: uwp
keywords: windows 10, uwp
---

# Bluetooth GATT Server

\[ Updated for UWP apps on Windows 10. For Windows 8.x articles, see the [archive](http://go.microsoft.com/fwlink/p/?linkid=619132) \]

**Important APIs**
- [**Windows.Devices.Bluetooth**](https://msdn.microsoft.com/library/windows/apps/Dn263413)
- [**Windows.Devices.Bluetooth.GenericAttributeProfile**](https://msdn.microsoft.com/library/windows/apps/Dn297685)


This article demonstrates Bluetooth Generic Attribute (GATT) Server APIs for Universal Windows Platform (UWP) apps, along with sample code for common GATT server tasks: 
- Define the supported services
- Publish server so it can be discovered by remote clients
- Advertise support for service
- Respond to read and write requests
- Send notifications to subscribed clients

## Overview
Windows usually operates in the client role. Nevertheless, many scenarios arise which require Windows to act as a Bluetooth LE GATT Server as well. Almost all the scenarios for IoT devices, along with most cross-platform BLE communication will require Windows to be a GATT Server. Additionally, sending notifications to nearby wearable devices has become a popular scenario that requires this technology as well.  
> Make sure all the concepts in the [GATT Client docs](gatt-client.md) are clear before proceeding.  

Server operations will revolve around the Service Provider and the GattLocalCharacteristic. These two classes will provide the functionality needed to declare, implement and expose a heirarchy of data to a remote device.

## Define the supported services
Your app may declare one or more services that will be published by Windows. Each service is uniquely identified by a UUID. 

### Attributes and UUIDs
Each service, characteristic and descriptor is defined by it's own unique 128-bit UUID.
> The Windows APIs all use the term GUID, but the Bluetooth standard defines these as UUIDs. For our purposes, these two terms are interchangeable so we'll continue to use the term UUID. 

If the attribute is standard and defined by the Bluetooth SIG-defined, it will also have a corresponding 16-bit short ID (e.g. Battery Level UUID  is 0000**2A19**-0000-1000-8000-00805F9B34FB and the short ID is 0x2A19). These standard UUIDs can be seen in [GattServiceUuids](https://msdn.microsoft.com/en-us/library/windows/apps/windows.devices.bluetooth.genericattributeprofile.gattserviceuuids.aspx) and [GattCharacteristicUuids](https://msdn.microsoft.com/en-us/library/windows/apps/windows.devices.bluetooth.genericattributeprofile.gattcharacteristicuuids.aspx).

If your app is implementing it's own custom service, a custom UUID will have to be generated. This is easily done in Visual Studio through Tools -> CreateGuid (use option 5 to get it in the "xxxxxxxx-xxxx-...xxxx" format). This uuid can now be used to declare new local services, characteristics or descriptors. 

### Build up the heirarchy of services and characteristics
The GattServiceProvider is used to create and advertise the root primary service definition.  Each service requires it's own ServiceProvider object that takes in a GUID: 

```csharp
GattServiceProviderResult result = await GattServiceProvider.CreateAsync(uuid);

if (result.Error == BluetoothError.Success)
{
    serviceProvider = result.ServiceProvider;
    // 
}
```
> Primary services are the top level of the GATT tree. Primary services contain characteristics as well as other services (called 'Included' or secondary services). 

Now, populate the service with the required characteristics and descriptors:

```csharp
GattLocalCharacteristicResult characteristicResult = await serviceProvider.Service.CreateCharacteristicAsync(uuid1, ReadParameters);
if (characteristicResult.Error != BluetoothError.Success)
{
    // An error occurred.
    return;
}
_readCharacteristic = characteristicResult.Characteristic;
_readCharacteristic.ReadRequested += ReadCharacteristic_ReadRequested;

characteristicResult = await serviceProvider.Service.CreateCharacteristicAsync(uuid2, WriteParameters);
if (characteristicResult.Error != BluetoothError.Success)
{
    // An error occurred.
    return;
}
_writeCharacteristic = characteristicResult.Characteristic;
_writeCharacteristic.WriteRequested += WriteCharacteristic_WriteRequested;

characteristicResult = await serviceProvider.Service.CreateCharacteristicAsync(uuid3, NotifyParameters);
if (characteristicResult.Error != BluetoothError.Success)
{
    // An error occurred.
    return;
}
_notifyCharacteristic = characteristicResult.Characteristic;
_notifyCharacteristic.SubscribedClientsChanged += SubscribedClientsChanged;
```
As shown above, this is also a good place to declare event handlers for the operations each characteristic supports.  To respond to requests correctly, an app must defined and set an event handler for each request type the attribute supports.  Failing to register a handler will result in the request being completed immediately with *UnlikelyError* by the system.

### Constant characteristics
Sometimes, there are characteristic values that will not change during the course of the app's lifetime. In that case, it is advisable to declare a constant characteristic to prevent unnecessary app activation: 

```csharp
byte[] value = new byte[] {0x21};
var constantParameters = new GattLocalCharacteristicParameters
{
    CharacteristicProperties = (GattCharacteristicProperties.Read),
    StaticValue = value.AsBuffer(),
    ReadProtectionLevel = GattProtectionLevel.Plain,
};

var characteristicResult = await serviceProvider.Service.CreateCharacteristicAsync(uuid4, constantParameters);
if (characteristicResult.Error != BluetoothError.Success)
{
    // An error occurred.
    return;
}
```
## Publish the service
Once the service has been fully defined, the next step is to publish support for the service. This informs the OS that the service should be returned when remote devices perform a service discovery.  You will have to set two properties - IsDiscoverable and IsConnectable:  

```csharp
GattServiceProviderAdvertisingParameters advParameters = new GattServiceProviderAdvertisingParameters
{
    IsDiscoverable = true,
    IsConnectable = true
};
serviceProvider.StartAdvertising(advParameters);
```
- **IsDiscoverable**: Advertises the friendly name to remote devices in the advertisement, making the device discoverable.
- **IsConnectable**:  Advertises a connectable advertisement for use in peripheral role.

> When a service is both Discoverable and Connectable, the system will add the Service Uuid to the advertisement packet.  There are only 31 bytes in the Advertisement packet and a 128-bit UUID takes up 16 of them!

## Respond to Read and Write requests
As we saw above while declaring the required characteristics, GattLocalCharacteristics have 3 types of events - ReadRequested, WriteRequested and SubscribedClientsChanged.

### Read
When a remote device tries to read a value from a characteristic (and it's not a constant value), the ReadRequested event is called. The characteristic the read was called on as well as args (containing information about the remote device) is passed to the delegate: 

```csharp
characteristic.ReadRequested += Characteristic_ReadRequested;
// ... 

async void ReadCharacteristic_ReadRequested(GattLocalCharacteristic sender, GattReadRequestedEventArgs args)
{
    // Our familiar friend - DataWriter.
    var writer = new DataWriter();
    // populate writer w/ some data. 
    // ... 

    var request = await args.GetRequestAsync();
    request.RespondWithValue(writer.DetachBuffer());
}
``` 

### Write
When a remote device tries to write a value to a characteristic, the WriteRequested event is called with details about the remote device, which characteristic to write to and the value itself: 

```csharp
characteristic.ReadRequested += Characteristic_ReadRequested;
// ...

async void WriteCharacteristic_WriteRequested(GattLocalCharacteristic sender, GattWriteRequestedEventArgs args)
{
    var request = await args.GetRequestAsync();
    var reader = DataReader.FromBuffer(request.Value);
    // Parse data as necessary. 

    if (request.Option == GattWriteOption.WriteWithResponse)
    {
        request.Respond();
    }
}
```
There are 2 types of Writes - with and without response. Use GattWriteOption (a property on the GattWriteRequest object) to figure out which type of write the remote device is performing. 

## Send notifications to subscribed clients
The most frequent of the GATT Server operations, notifications perform the critical function of pushing data to the remote devices. Sometimes, you'll want to notify all subscribed clients but othertimes you may want to pick which devices to send the new value to: 

```csharp
async void NotifyValue()
{
    var writer = new DataWriter();
    // Populate writer with data
    // ...
    
    await notifyCharacteristic.NotifyValueAsync(writer.DetachBuffer());
}
```

When a new device subscribes for notifications, the SubscribedClientsChanged event gets called: 

```csharp
characteristic.SubscribedClientsChanged += SubscribedClientsChanged;
// ...

void _notifyCharacteristic_SubscribedClientsChanged(GattLocalCharacteristic sender, object args)
{
    List<GattSubscribedClient> clients = sender.SubscribedClients;
    // Diff the new list of clients from a previously saved one 
    // to get which device has subscribed for notifications. 

    // You can also just validate that the list of clients is expected for this app.  
}

```
> Note that an application can get the maximum notification size for a particular client with the MaxNotificationSize property.  Any data larger than the maximum size will be truncated by the system.
