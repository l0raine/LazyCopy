This minifilter driver ([MSDN](https://msdn.microsoft.com/en-us/library/windows/hardware/ff540402%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396)) intercepts operations on the special reparse point files ([MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365503(v=vs.85).aspx)). If such file is opened for the first time, driver downloads its content from the remote location.

It's similar to the [Git Virtual File System](https://github.com/Microsoft/GVFS) project from Microsoft. And lacks official support.

Content can be downloaded from:
* `Local storage`.
* `Network share`. If driver cannot open the file, it asks user-mode service to open that file, and then downloads its content.
* `URI`. Driver asks the user-mode service to download that file.

You can easily extend the user-mode service to support more types.

Short demo
-------

![img](https://github.com/aleksk/LazyCopy/blob/master/demo.gif)

Prerequisites
-------

1. Windows 7+
2. [Visual Studio 2015 & WDK 10](https://developer.microsoft.com/en-us/windows/hardware/windows-driver-kit)
3. [WiX toolset](https://wix.codeplex.com/releases/view/624906)

Folder structure
-------

- `Driver`
  - `LazyCopyDriver`        - Minifilter driver.
  - `LazyCopyDriverInstall` - Generates driver installation package.
  - `DriverClientLibrary`   - C# library allows interacting with drivers via the communication ports ([MSDN](https://msdn.microsoft.com/en-us/library/windows/hardware/ff541931(v=vs.85).aspx)).
  - `LazyCopyDriverClient`  - LazyCopy C# driver client based on the `DriverClientLibrary`.
- `ToolsAndLibraries`
  - `Utilities`             - Contains shared helper classes.
  - `EventTracing`          - Allows collecting and decoding of the ETW ([MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/bb968803(v=vs.85).aspx)) events generated by the driver.
  - `SampleClient         ` - A basic console C# application that can create files that are understood by the driver.
- `Service\LazyCopySvc`     - A user-mode system service that manages the lifetime and configuration of the driver. It can also open files on behalf of the currently logged in user, or download them per driver request.
- `Setup`                   - [WiX](http://wixtoolset.org/) installation package to install driver, service and the client applications.

Compilation
-------

1. Make sure you have the latest [WDK](https://developer.microsoft.com/en-us/windows/hardware/windows-driver-kit) installed.
2. Open the `LazyCopyDriver` project properties, and make sure the `General > Target Platform Version` value corresponds to the WDK version you installed.
3. (Optionally) Configure driver test signing in the `Properties > Driver Signing > General`.
4. Make sure the solution is compiled for your architecture (`Main menu > Build > Configuration Manager`).

Driver signing
-------

* Get a code signing certificate: [MSDN](https://msdn.microsoft.com/en-us/library/windows/hardware/hh801887.aspx)
* Get the cross-certificate for it ([MSDN](https://msdn.microsoft.com/en-us/library/windows/hardware/dn170454(v=vs.85).aspx)). You may want to use the [VeriSign Cross-Certificate](http://go.microsoft.com/fwlink/p/?linkid=321787).
* Sign the driver and, optionally, other binaries.
  If you purchased a VeriSign certificate, you can use the following command to sign the driver in the post-build step:
```
signtool sign /v /s my /n "<YOUR_NAME>" /sha1 "<YOUR_CERT_THUMBNAIL>" /ac "<PATH_TO_CROSS_CERT>" /t http://timestamp.verisign.com/scripts/timestamp.dll "$(TargetPath)\LazyCopyDriver.sys"
&
signtool sign /v /s my /n "<YOUR_NAME>" /sha1 "<YOUR_CERT_THUMBNAIL>" /ac "<PATH_TO_CROSS_CERT>" /t http://timestamp.verisign.com/scripts/timestamp.dll "$(TargetPath)\LazyCopyDriver.cat"
```
For example:
```
signtool sign /v /s my /n "Contoso Org" /sha1 "CAFEBEBE0123456701BE7F9D3BBDFBB230233386" /ac "c:\temp\VeriSign_Cross_Sign.cer" /t http://timestamp.verisign.com/scripts/timestamp.dll "$(TargetPath)\LazyCopyDriver.sys"
```

Installation
-------

* Allow Windows to load drivers signed with the test certificates:
   1. Open CMD as Admin and type ([MSDN](https://msdn.microsoft.com/en-us/library/windows/hardware/ff553484(v=vs.85).aspx)): `bcdedit -set TESTSIGNING ON`
   2. Reboot.
* Compile the entire solution in the Visual Studio <i>for your architecture</i>. Make sure to choose the valid `Target Platform Version` in the `LazyCopyDriver` project settings.
* You can manually install the driver by right clicking on the `.inf` file and choosing `Install`.
* Check that LazyCopyDriver appeared in the `fltmc` command output ([MSDN](https://msdn.microsoft.com/en-us/library/windows/hardware/ff548166(v=vs.85).aspx)).
<br/>From the Admin CMD:
```
> fltmc
Filter Name                     Num Instances    Altitude    Frame
------------------------------  -------------  ------------  -----
LazyCopyDriver                          7        180610        0
FileInfo                                8        45000         0
```
Depending on the load type specified in the `.inf` file, it might not be automatically loaded. You can do it manually:
```
> fltmc load lazycopydriver
```
* Install and start the LazyCopySvc. It is optional and needed, if you want to have a custom download logic (for example, being able to download files via HTTP) or share the stub files over the network.
```
> sc create LazyCopySvc binPath="<Absolute_path_to_LazyCopySvc.exe>" DisplayName="LazyCopySvc"
> sc start LazyCopySvc
```

Trying it out
-------

Create an empty file that will be fetched on the first access (admin permissions are required):
```
bin\SampleClient\CreateLcFile.exe < original file >  < new empty file >

.\CreateLcFile.exe "\\build\latest\contoso.dll"  "c:\temp\contoso.dll"
.\CreateLcFile.exe "http://www.contoso.org/"     "c:\temp\index.html"
.\CreateLcFile.exe "d:\data\file_with_data.txt"  "c:\temp\yet_empty_file.txt"
```

Want to reuse project files?
-------

* Change the name ([MSDN](Driver/LazyCopyDriver/LazyCopyDriver.inf)) of the driver and rename binaries.
* Generate ([MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/aa385638(v=vs.85).aspx)) a new GUID for the [ETW provider](Driver/LazyCopyDriver/LazyCopyEtw.mc) and re-create header:
```
mc.exe -z LazyCopyEtw -n -km LazyCopyEtw.mc
```
* Contact Microsoft and get the following values:
  - [LC_REPARSE_GUID](Driver/LazyCopyDriver/LazyCopyDriver.c) - from [MSDN](https://msdn.microsoft.com/en-us/library/windows/hardware/dn641624(v=vs.85).aspx)
  - [Instance1.Altitude](Driver/LazyCopyDriver/LazyCopyDriver.inf) - from [MSDN](https://msdn.microsoft.com/en-us/library/windows/hardware/dn508284(v=vs.85).aspx)
* (Optional) Change the reparse point tag: [LC_REPARSE_TAG](Driver/LazyCopyDriver/Globals.h).
* Update the client code with the new reparse GUID and tag: [LazyCopyFileHelper.cs](Driver/LazyCopyDriverClient/LazyCopyFileHelper.cs).
* (Optional) Disable the sharing access override in the [Operations.c](Driver/LazyCopyDriver/Operations.c) (see the `PreCreateOperationCallback` method).
* Make sure that all 'TODO:' items are addressed in the code.
