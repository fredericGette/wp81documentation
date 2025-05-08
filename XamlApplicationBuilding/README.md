# How to build a Windows Phone XAML application

## Requirements

- Visual Studio 2015 with the Windows Phone SDK.
  ![visualStudio](Capture00.PNG)

## Debuging

Meaning of the [4 counters](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.debugsettings.enableframeratecounter?view=winrt-26100) displayed at the right side of the screen (from top to bottom):  

| App fps |	App CPU	| Sys fps | Sys CPU |
|:-:|:-:|:-:|:-:|
| The app's UI thread frame rate, in frames per second.	| The CPU usage of the app's UI thread per frame, in milliseconds. | The system-wide composition engine frame rate, in frames per second. This is typically pegged to 60. |	The system-wide overall CPU usage of the composition thread per frame, in milliseconds. |
