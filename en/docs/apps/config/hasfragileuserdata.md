The [`hasFragileUserData`](https://developer.android.com/guide/topics/manifest/application-element#fragileuserdata) flag can be added to the `application` node of `AndroidManifest.xml`. If its value is `true,` then when the user uninstalls the app, a prompt will be shown to the user asking him whether to keep the app's data.

```
<application
	...
    android:hasFragileUserData="true" tools:targetApi="q">
...
</application>
```

The flag engages regardless of `MANAGE_EXTERNAL_STORAGE` or scoped storage on target sdk android `>=10`, don't see any checks in [`PackageParser`](https://cs.android.com/android/platform/superproject/+/android-10.0.0_r30:frameworks/base/core/java/android/content/pm/PackageParser.java;l=3720), [`ApplicationInfo.hasFragileUserData()`](https://cs.android.com/android/platform/superproject/+/android-10.0.0_r30:frameworks/base/core/java/android/content/pm/ApplicationInfo.java;l=1845) or while creating the [`UninstallAlertDialogFragment`](https://cs.android.com/android/platform/superproject/+/android-10.0.0_r30:frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/handheld/UninstallAlertDialogFragment.java;l=167).

To be able to add the flag in `AndroidManifest.xml` requires the `compileSdkVersion` to be `>= 29`, otherwise android studio (`aapt`) does not let you compile the apk and shows the `Unknown Attribute` warning in editor and throws `AAPT: error: attribute android:hasFragileUserData not found.` exception when building.

Whether the flag is present or not in the `AndroidManifest.xml` of apps in android `>= 10` should not affect any files under `/storage/emulated/0/` created by the app, basically they will not be automatically deleted on app uninstall. However, if the flag is added, then data stored in app private directories in `/storage/emulated/0/Android/{data,media,obb}/<package_name>` and `/data/data/<package_name>` may not be deleted if the user enables the `Keep app data` toggle in the uninstall dialog shown if app is uninstalled from Android Settings **only**. The dialog is not shown for play store, `adb`, etc and all data is removed. The documentation for `hasFragileUserData` should mention this. There is an xda article on this as well [here](https://www.xda-developers.com/android-10-manifest-flag-developers-retain-app-data-before-uninstalling/).


Another thing I realized and then investigated was the potential security risk. If you enable the toggle during app uninstallation to keep its data, the app will still show in the Android Settings App list and will be in a `semi-uninstall` state. You can **fully** uninstall the app if you wish by pressing the uninstall button again. This is likely done to preserve package information as well like APK signature. Basically, users shouldn't be allowed to install an app, like from like playstore, then uninstall the app while keeping its data by enabling the toggle, then building a new custom APK themselves with the same package name and install it again to get access to the private data of the app. That is a security risk. Android package manager by default (unless custom roms) doesn't even allow version downgrades of apps, let alone package signature mismatch. If you attempt to install the app again that is in the `semi-uninstall` state with an APK with a different signature, the installation will fail with errors like `The APK failed to install. Error: INSTALL_FAILED_UPDATE_INCOMPATIBLE: Package com.termux signatures do not match previously installed version; ignoring!` errors. Basically, from a user point of view, this means that if the app was installed from playstore, you will only be able to get access to the previous install data, if you install the same app again from playstore (assuming playstore ris the only place you can get the app with a specific signature). In the case of Termux, the Playstore, F-Droid and Debug/Github action builds all have different signing keys/APK signatures, so users can't shift from one source to another while keeping their app data, since installation will fail if they try to do so. They can only reinstall the APK if it has the same source as the one installed previously.


Now here it gets even more interesting, if the user tries to uninstall an app on Android `10` (not `11`) from Android Settings, which has **just** declared the `hasFragileUserData` flag in the `AndroidManifest.xml` and the device has an external removable sd card set up as [portable storage](https://source.android.com/devices/storage), then the android `packageinstaller` will crash internally and nothing will be shown to the user and app uninstall will fail as well, even without the `Keep app data` diaglog being shown. The stacktrace is mentioned below, it apparently happens because the sd card UUID string is not a valid format when [StorageManager](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r3:frameworks/base/core/java/android/os/storage/StorageManager.java;l=2654) tries to calculate the private app data size on the sd card. with At least, it does fail on Android `10` virtual device (avd), even trying it after ejecting the sd card does not work for some reason. There is an issue reported on issuetracker for this [here](https://issuetracker.google.com/issues/148573843) and an old but similar one [here](https://issuetracker.google.com/issues/62982912
). The issue was fixed in android `11` by the [4044831](https://cs.android.com/android/_/android/platform/frameworks/base/+/4044831837cb3401e262db3614ac41fb343aebf4) commit by using [ApplicationInfo.storageUuid](https://developer.android.com/reference/android/content/pm/ApplicationInfo#storageUuid) instead of looping on all storage volumes. I tried this on a physical android `10` device (Xiaomi 10t lite) but it didn't even show the uninstall dialog with the `Keep user data` toggle and directly uninstalled the app, maybe because of this bug, so apparently all manufactures have not enabled this functionality either.

So @termux has to decide if this flag should even be added, even for `android-10` branch. Since users can't move from one source to another with this flag, and it will prevent users from even uninstalling the app from Android Settings on Android `10`, which would increase bug reports posted, is it even worth it? I don't know if its possible to restrict this flag for android `>= 11` devices only for the same release APK, the documentation does not mention anything from my quick look.

##### Expert Tip

If you are writing persistent malware apps for android `10` devices, use the `hasFragileUserData` flag and hope your user has a portable sd card installed and does not know how to uninstall apps from playstore or with `adb`, which would be most users. Enjoy!


##### Internal details of `hasFragileUserData`

If you ask the dialog to preserve the data, then [`DELETE_KEEP_DATA`](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r3:frameworks/base/core/java/android/content/pm/PackageManager.java;l=1597) is passed to [`PackageInstallerService.uninstall()`](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r3:frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/UninstallUninstalling.java;l=101). Ultimately, this reaches [`removePackageDataLIF()`](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r3:out/soong/.intermediates/frameworks/base/services/core/services.core.unboosted/android_common/xref/srcjars.xref/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java;l=18817) which calls the native [`destroyAppData()`](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/installd/InstalldNativeService.cpp;l=716) to do the actual deletion, which is only deletes app user specific paths.


##### Stacktrace for android 10 uninstall failure if app has declared `hasFragileUserData`

```
com.google.android.packageinstaller E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.google.android.packageinstaller, PID: 7079
    java.lang.RuntimeException: Unable to start activity ComponentInfo{com.google.android.packageinstaller/com.android.packageinstaller.UninstallerActivity}: java.lang.IllegalArgumentException: Invalid UUID string: 1513-2403
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3270)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3409)
        at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:83)
        at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
        at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
        at android.os.Handler.dispatchMessage(Handler.java:107)
        at android.os.Looper.loop(Looper.java:214)
        at android.app.ActivityThread.main(ActivityThread.java:7356)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
     Caused by: java.lang.IllegalArgumentException: Invalid UUID string: 1513-2403
        at java.util.UUID.fromString(UUID.java:194)
        at android.os.storage.StorageManager.convert(StorageManager.java:2290)
        at com.android.packageinstaller.handheld.UninstallAlertDialogFragment.getAppDataSizeForUser(UninstallAlertDialogFragment.java:78)
        at com.android.packageinstaller.handheld.UninstallAlertDialogFragment.getAppDataSize(UninstallAlertDialogFragment.java:114)
        at com.android.packageinstaller.handheld.UninstallAlertDialogFragment.onCreateDialog(UninstallAlertDialogFragment.java:179)
        at android.app.DialogFragment.onGetLayoutInflater(DialogFragment.java:417)
        at android.app.Fragment.performGetLayoutInflater(Fragment.java:1351)
        at android.app.FragmentManagerImpl.moveToState(FragmentManager.java:1303)
        at android.app.FragmentManagerImpl.addAddedFragments(FragmentManager.java:2431)
        at android.app.FragmentManagerImpl.executeOpsTogether(FragmentManager.java:2210)
        at android.app.FragmentManagerImpl.removeRedundantOperationsAndExecute(FragmentManager.java:2166)
        at android.app.FragmentManagerImpl.execPendingActions(FragmentManager.java:2067)
        at android.app.FragmentManagerImpl.dispatchMoveToState(FragmentManager.java:3057)
        at android.app.FragmentManagerImpl.dispatchActivityCreated(FragmentManager.java:3004)
        at android.app.FragmentController.dispatchActivityCreated(FragmentController.java:184)
        at android.app.Activity.performCreate(Activity.java:7809)
        at android.app.Activity.performCreate(Activity.java:7791)
        at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1299)
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3245)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3409) 
        at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:83) 
        at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135) 
        at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95) 
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016) 
        at android.os.Handler.dispatchMessage(Handler.java:107) 
        at android.os.Looper.loop(Looper.java:214) 
        at android.app.ActivityThread.main(ActivityThread.java:7356) 
        at java.lang.reflect.Method.invoke(Native Method) 
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492) 
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
```