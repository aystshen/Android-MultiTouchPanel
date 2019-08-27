# Dual screen touch --- Implement the specified touch as the secondary touch

In the dual-screen display device, sometimes the main and secondary screens are equipped with a touch screen, and the main and auxiliary screens are required to touch each other without interference.

Android's existing framework already supports the logic of the secondary input device, but by default all the external hot-swappable devices are uniformly designated as the secondary input device. This logic can be supported by default if it is an i2c plus a usb touch, usb Touch is the secondary touch.

However, sometimes we are a pair of dual i2c or dual usb, we need to modify the existing logic, the program is as follows:

> Configure the sub-screen Touch by the property: device name, pid&vid, usb port, get the device name of the input device, pid&vid, usb port and the property value in EventHub. If it is the device in the configuration, mark it as the extern input device. 

## Integration
1. Apply multi-touch.patch.
2. Recompile Android.

## Usage 
Property name：ro.input.external  
Property value：

Device Type | Format | Example
---|---|---
usb | vid:pid | 222a:0001
usb | usb port | usb-1.4
i2c | device name | Hanvon electromagnetic pen

It is also possible to configure multiple devices at the same time, with each property value separated by ",".

Example：
```
ro.input.external=222a:0001,Hanvon electromagnetic pen,usb-1.4
```

The above property configuration "vid=222a, pid=0001" usb touch and the device name "Hanvon electromagnetic pen" i2c touch and the usb port 1.4 touch are the secondary screen touch, the other unconfigured defaults to the main screen touch.

## Developed By
* ayst.shen@foxmail.com

## License
```
Copyright 2019 Bob Shen

Licensed under the Apache License, Version 2.0 (the "License"); you may 
not use this file except in compliance with the License. You may obtain 
a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software 
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT 
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the 
License for the specific language governing permissions and limitations 
under the License.
```