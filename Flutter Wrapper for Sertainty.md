# Flutter Wrapper for Sertainty

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://tfs.champsoft.com/tfs/DefaultCollection/MYOU_Mobile/_git/Sertainty_flutter_Wrapper)

Flutter wrapper for sertainty using Sertainty SDK and Dart FFI.

  - Support for both Android and iOS
  - Encryption and Decryption and SMEX working and tested

# Flutter App Integration Guide

###### 1. Add flutter wrapper as plugin to you project pubspec.yaml file , Path should point to Sertainty Wrapper plugin’s pubspec.yaml as below.

```sh 
dependencies:
  flutter:
    sdk: flutter
  sertainty_wrapper:
    path: ./sertainty_wrapper
```

##### 2. Add Sertainty license, boot.ini , sertainty_android.rsf and sertainty_ios.rsf to project as assets folder

![img ss](https://github.com/tharukachampit/sertainty_flutter_docs/blob/main/images/Screenshot%202021-01-19%20at%2020.55.36.png)

For android it's easy. We can just add the plugin and use it without any additional steps, but for ios below steps are required.

##### 3.Add SertaintyCore.framework as embedded framework to Framework, Libraries, and Embedded Content on XCode.

![Screenshot 2021-01-19 at 21.12.46](https://github.com/tharukachampit/sertainty_flutter_docs/blob/main/images/Screenshot%202021-01-19%20at%2021.12.46.png)



##### 4.We need to add a build phase run script in order to copy SertaintyCore.framework file into the build file. Otherwise will get an error when initialising the app

 (Error : dylib not found or image not found)

Script:

```sh
install_name_tool -change SertaintyCore.framework/SertaintyCore @executable_path/Frameworks/SertaintyCore.framework/SertaintyCore $TARGET_BUILD_DIR/$TARGET_NAME.app/$TARGET_NAME
```

![Screenshot 2021-01-19 at 21.20.35](https://github.com/tharukachampit/sertainty_flutter_docs/blob/main/images/Screenshot%202021-01-19%20at%2021.20.35.png)



##### 5. With the latest macOS and iOS updates we can not embed .framework files that support both emulator and real device architecture from same file therefore we will have to use .XCFramework file but Sertainty still not supported for that so you’ll get an error as below

(Error: ios simulator + arm64 architecture not supported ) 

> Fix: On XCode build settings add Validate workspace to ‘yes’

 ![Screenshot 2021-01-19 at 21.38.16](https://github.com/tharukachampit/sertainty_flutter_docs/blob/main/images/Screenshot%202021-01-19%20at%2021.38.16.png)



## Sertainty Wrapper Usage Guide

Before using Sertainty Wrapper we need to set up necessary files and initialize the sertainty library.

##### 1. Copy all necessary assets to the 'app_root' folder Using a given helper method.

```dart
var _callStatusHandle;
var _buffer;
var _appHandle;
bool _hasError = false;
final String _licenseFile = 'sertainty.lic';
final String _appKey = 'SertintyONE';
final String _prefix = 'SampleJNI';
final String _version = '1.0';
String _appRoot;
String get _downloadsPath => _appRoot + '/Sertainty/downloads/';
String get _decryptedFilesPath => _appRoot + '/Sertainty/example/temp/';

Future<bool> setupSertaintySDk() async {
  String message = await installAssets();
  _appRoot = uxpsys_getAppRoot();
  _callStatusHandle = uxpsys_newCallStatusHandle();
  _buffer = uxpba_newHandle();
  _appHandle = uxpfile_newHandle();
  if (message != null) {
    return true;
  } else {
    return false;
  }
}
```



Note * : 

> Required assets files should be in the project assets folder (Read Flutter App Integration guide : step 2 ).

##### 2. Install Sertainty developer License using " uxpsys_installLicense() " method in Sertainty Wrapper.

Example : 

```dart
Future<bool> installLicense(String fileName) async {
  File newLicenseFile;
  int status = 1;
  ByteData pathData =
      await rootBundle.load("assets/Sertainty/home/$fileName.lic");
  newLicenseFile = new File(_appRoot + "/sertainty.lic");
  if (!newLicenseFile.existsSync()) {
    await newLicenseFile.writeAsBytes(pathData.buffer
        .asUint8List(pathData.offsetInBytes, pathData.lengthInBytes));
    status = uxpsys_installLicense(_buffer, newLicenseFile.absolute.path);
  }
  if (status == 1) {
    return true;
  }
  return false;
}
```



##### 3. Initialize Sertainty library using " uxpsys_initLibrary() "

```dart
Future<bool> initializeLibrary() {
  var emptyArray = [];
  int ret = uxpsys_initLibrary(
    _buffer,
    0,
    emptyArray.toString(),
    _licenseFile,
    _appKey,
    _prefix,
    _version,
  );
  return Future.value(ret == 0 ? false : true);
}
```

##### 4.Next we can use Sertainty Library for file encryption and decryption and other sertainty use cases

* You can find an example app inside Sertainty Wrapper plugin

Example :

```dart
String getDecryptedUxpFileUri(
    String uxpFileName, String fileExtension, credentials) {
  _appHandle = uxpfile_newHandle();

  String uxpFile = uxpFileName + ".uxp";
  String decryptedFileWithExt = uxpFileName + '.$fileExtension';
  String uxpFileUri = _downloadsPath + uxpFile;
  String decryptedFileUri = _decryptedFilesPath + decryptedFileWithExt;
  print(uxpFileUri);
  print(decryptedFileUri);
  uxpfile_openFile(_appHandle, uxpFileUri, 2);
  checkError(_appHandle);
  if (_hasError) {
    return null;
  }

  bool isAuthorized = _checkAutharization(credentials);
  if (!isAuthorized) {
    return null;
  }
  uxpfile_openVirtualFile(_appHandle, 'file', 1);
  checkError(_appHandle);
  if (_hasError) {
    return null;
  }
  uxpfile_exportVirtualFile(_appHandle, 'file', decryptedFileUri, 0x00001);
  checkError(_appHandle);
  if (_hasError) {
    return null;
  }
  return decryptedFileUri;
}

// ignore: missing_return
bool _checkAutharization(credentials) {
  var done = 0;
  while (done == 0) {
    var authStatus = uxpfile_authenticate(_appHandle, 0);

    switch (authStatus) {
      case StatusAuthorized:
        print("You're authorized\n");
        done = 1;
        return true;

      case StatusNotAuthorized:
        done = 1;
        return false;

      case StatusChallenged:
        for (int i = 0; i < uxpfile_getChallengeCount(_appHandle); i++) {
          print(uxpfile_getChallengeCount(_appHandle));
          var ch = uxpfile_getChallenge(_appHandle, i);
          var prompt = uxpba_newHandle();

          uxpch_getPrompt(ch, prompt);

          var challenge = uxpba_getData(prompt);

          print("Enter $challenge  : ${credentials[challenge]}");

          uxpch_setValueString(ch, credentials["$challenge"]);

          uxpfile_addResponse(_appHandle, ch);

          uxpch_freeHandle(ch);
        }
        break;

      default:
        break;
    }
  }
}
```

 
