![2230](img/system-ui/2230.png)


apk推送到设备上

```
adb push SystemUI.apk /system/system_ext/priv-app/SystemUI/

adb shell killall com.android.systemui
```

如果SystemUI不能正常起来，则需要重启一下设备


```
adb reboot
```