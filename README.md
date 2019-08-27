# 双屏异触 --- 实现指定触摸为副屏触摸功能

在双屏异显产品中，有时候主副屏都带有触摸屏，并且要求主副屏触摸各自操作互不干扰。

Android 现有框架中已经支持副输入设备的逻辑，只是默认将所有的外部热插拔设备统一指定为副输入设备，这种逻辑我们如果是一个 i2c 加上一个 usb 触摸那么默认就可以支持，usb触摸就是副tp。

但，有时候我们是双 i2c 或双 usb 的搭配，我们就需要改造现有逻辑，方案如下：  

> 通过属性配置副屏 tp 的： 设备名、pid&vid、usb端口，在 EventHub 中获取输入设备的设备名、pid&vid、usb端口与属性值进行对比，如果是配置中的设备就将其标记为副输入设备。

## 实现 
```diff
diff --git a/frameworks/native/services/inputflinger/EventHub.cpp b/frameworks/native/services/inputflinger/EventHub.cpp
old mode 100644
new mode 100755
index 2bcc5c7..1542a7b
--- a/frameworks/native/services/inputflinger/EventHub.cpp
+++ b/frameworks/native/services/inputflinger/EventHub.cpp
@@ -64,6 +64,11 @@
 #define INDENT2 "    "
 #define INDENT3 "      "
 
+// for multi touch panel
+#define DEVICE_MATCH_METHOD_MAX 10
+#define USB_LOCATION_MATCH_START 13 //"usb-ff540000."
+#define USB_LOCATION_MATCH_LEN 7 //"usb-1.1"
+
 namespace android {
 
 static const char *WAKE_LOCK_ID = "KeyEvents";
@@ -1184,17 +1189,17 @@ status_t EventHub::openDeviceLocked(const char *devicePath) {
     int32_t deviceId = mNextDeviceId++;
     Device* device = new Device(fd, deviceId, String8(devicePath), identifier);
 
-    ALOGV("add device %d: %s\n", deviceId, devicePath);
-    ALOGV("  bus:        %04x\n"
+    ALOGI("add device %d: %s\n", deviceId, devicePath);
+    ALOGI("  bus:        %04x\n"
          "  vendor      %04x\n"
          "  product     %04x\n"
          "  version     %04x\n",
         identifier.bus, identifier.vendor, identifier.product, identifier.version);
-    ALOGV("  name:       \"%s\"\n", identifier.name.string());
-    ALOGV("  location:   \"%s\"\n", identifier.location.string());
-    ALOGV("  unique id:  \"%s\"\n", identifier.uniqueId.string());
-    ALOGV("  descriptor: \"%s\"\n", identifier.descriptor.string());
-    ALOGV("  driver:     v%d.%d.%d\n",
+    ALOGI("  name:       \"%s\"\n", identifier.name.string());
+    ALOGI("  location:   \"%s\"\n", identifier.location.string());
+    ALOGI("  unique id:  \"%s\"\n", identifier.uniqueId.string());
+    ALOGI("  descriptor: \"%s\"\n", identifier.descriptor.string());
+    ALOGI("  driver:     v%d.%d.%d\n",
         driverVersion >> 16, (driverVersion >> 8) & 0xff, driverVersion & 0xff);
 
     // Load the configuration file for the device.
@@ -1357,10 +1362,35 @@ status_t EventHub::openDeviceLocked(const char *devicePath) {
     }
 
     // Determine whether the device is external or internal.
-    if (isExternalDeviceLocked(device)) {
+    if((device->classes & 0x04) == INPUT_DEVICE_CLASS_TOUCH) {
+        int count = 0;
+        char flag[DEVICE_MATCH_METHOD_MAX][PROPERTY_VALUE_MAX];
+        char value[PROPERTY_VALUE_MAX] = {0};
+    
+        property_get("ro.input.external", value, "");
+
+        if (isExternalDeviceLocked(device)) {
+            sprintf(flag[count++], "%04x:%04x", identifier.vendor, identifier.product);
+            if (identifier.location.length() >= USB_LOCATION_MATCH_START+USB_LOCATION_MATCH_LEN) {
+                strncpy(flag[count++], identifier.location.string()+USB_LOCATION_MATCH_START, USB_LOCATION_MATCH_LEN);
+            }
+        } else {
+            sprintf(flag[count++], "%s", device->identifier.name.string());
+        }
+
+        for (int i=0; i<count; i++) {
+            ALOGI("openDeviceLocked:%d, value=%s flag=%s\n", __LINE__, value, flag[i]);
+            if (strstr(value, flag[i])) {
+                device->classes |= INPUT_DEVICE_CLASS_EXTERNAL;
+                ALOGI("openDeviceLocked:%d, name:\"%s\" id:%d device_class:%x vid:%04x pid:%04x is external input device\n",
+                    __LINE__, device->identifier.name.string(), device->id, device->classes, identifier.vendor, identifier.product);
+                break;
+            }
+        }
+    } else {
         device->classes |= INPUT_DEVICE_CLASS_EXTERNAL;
     }
-
+    
     if (device->classes & (INPUT_DEVICE_CLASS_JOYSTICK | INPUT_DEVICE_CLASS_DPAD)
             && device->classes & INPUT_DEVICE_CLASS_GAMEPAD) {
         device->controllerNumber = getNextControllerNumberLocked(device);

```

## 属性配置格式说明 
属性名：ro.input.external  
属性值：

设备类型 | 格式 | 例如
---|---|---
usb | vid:pid | 222a:0001
usb | usb端口 | usb-1.4
i2c | 设备名 | Hanvon electromagnetic pen

也可以同时配置多个设备，各属性值之间用“,”隔开。

例如：
```
ro.input.external=222a:0001,Hanvon electromagnetic pen,usb-1.4
```

以上属性配置“vid=222a,pid=0001”的 usb tp 和设备名为“Hanvon electromagnetic pen”的 i2c tp 以及 usb 端口为 1.4 的 tp 为副屏 tp，其它未配置的都默认为主屏 tp。