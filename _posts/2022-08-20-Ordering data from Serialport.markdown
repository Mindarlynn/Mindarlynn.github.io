---
layout: post
title:  "Ordering data from Serial port"
date:   2022-08-20 03:30:00 +0900
categories: C# Networking
---
When using Arduino, or other embedded boards, we receive data through a serial port.

Suppose that this embedded board is always in operation. It will send data to the serial port as soon as it is powered on, and all data will disappear until we turn on the program that receives the value from the serial port.

Assuming that the embedded board sends a serialized 20byte structure, running the program after it sends by 10byte would result in collecting the lower 10byte of the first data and the upper 10byte of the second data, although we expected the data of the serialized 20byte.

We need to convert the serialized data sent from the embedded into the correct structure. I'm going to suggest a way to use it at this time.

The first is to add a few bytes of key to the end of the structure you are trying to send.
For example, we declare char[2] at the end of the structure and insert values that we can identify that will never be assigned to other variables in the structure. In this example, we will use "hi".

```csharp
struct awesome_struct{
    ...
    char key[2] = "hi";
}
```
We will send a structure like this on the embedded board, which will have a total size of 22bytes, including data 20byte and key 2byte.

We will find the key "hi" in the data received as a serial, and then we will truncate the 22 bytes of data containing the key and convert it into a structure.

```csharp
SerialPort _serialPort
ConcurrentQueue<byte> _receivingBuffer
bool _isOpen = false;

Thread _packetProcessThread;

public initializer(){
    ...
    // Assign callback to DataReceived event
    _serialPort.DataReceived += DataReceived;
    _packetProcessThread = new Thread(ProcessPacket);
    _packetProcessThread.Start();
    _isOpen = true;
}

void DataReceived(object sender, SerialDataReceivedEventArgs e){
    var serialPort = (SerialPort)sender;
    try{
        var size = serial.BytesToRead;
        var buffer = new byte[size];

        // Read data from serial port and put all data into the receiving queue.
        serial.Read(buffer, 0, size);
        buffer.ToList().ForEach(_receivingBuffer.Enqueue);
    }
    catch(Exception ex){
        Console.WriteLine(ex.Message);
        // clean-up and reopen serial port here
    }
}

void ProcessPacket(){
    while(_isOpen){
        // Total size of struct is 22byte
        if (receivingBuffer.Count < 22) continue;

        // Add each byte to aggregator and we will find "hi" in aggregator.
        var aggregator = new Queue<byte>();

        while(true){
            // Consider the case where the first 2byte of the receiving buffer is "hi" due to the loss of data loss is also considered.
            // 0x68 = 'h'
            // 0x69 = 'i'
            if (aggregator.Count >= 2 &&
                aggregator.ElementAt(aggregator.Count - 2) == 0x68
                aggregator.ElementAt(aggregator.Count - 1) == 0x69)
                break;

            // If "hi" is not found yet, take the byte from the receiving buffer and enqueue it to the aggregator.
            _receivingBuffer.TryDequeue(out var b);
            aggregator.Enqueue(b);        
        }

        // If the data containing "hi" is less than the total size of 22byte, the data is discarded.
        if (aggregator.Count < 22) continue;

        // If the data containing "hi" is greater than the total size of 22byte, the aggregator's front data is discarded to match 22byte.
        // If it is larger than 22byte, most of the previous data is lost and only 'i' remains.
        while (aggregator.Count > 22) aggregator.Dequeue();

        var packet = aggregator.ToArray();

        // Convert packet to struct here.
    }
}
```
In this way, the received data may be stably converted into a structure without being pushed back.

I used this method to perform RF communication through Arduino. Through this code, it was possible to receive the value transmitted from the counterpart Arduino very stably. You can see how I used this code: [here]


[here]: https://github.com/cande-cansat/SatGS/blob/develop/SatGS.Communication/SerialReceiver.cs
