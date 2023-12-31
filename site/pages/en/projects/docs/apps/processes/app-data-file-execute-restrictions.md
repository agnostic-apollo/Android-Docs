# App Data File Execute Restrictions

The private app data files on Android normally exists under the `/data/data/<package_name>` (for user `0`) or `/data/user/<user_id>/<package_name>` (for all users) directory if app is installed on internal sd. If app is installed on a removable/portable volume/sd card being used as [adoptable storage](https://source.android.com/docs/core/storage/adoptable), then the path should be `/mnt/expand/<volume_uuid>/user/<user_id>/<package_name>`.

To execute or load shared libraries under the app data directory requires the SeLinux [process context type](../../os/selinux/context-types.md#process-context-types) assigned to the app to have the execute permission for the [file context type](../../os/selinux/context-types.md#file-context-types) assigned to the files in the app data directory. The `file` object `execute` permission is required to `dlopen()` shared libraries in app data directory, like with [`System.load(String)`](https://developer.android.com/reference/java/lang/System#load(java.lang.String)) in Java or for when shared library dependencies of ELF binaries are loaded by [dynamic linker](https://en.wikipedia.org/wiki/Dynamic_linker). The `execute_no_trans` permission is required to `exec()` files in app data directory. 

- https://selinuxproject.org/page/NB_ObjectClassesPermissions#File_Object_Classes
- https://man7.org/linux/man-pages/man3/dlopen.3.html
- https://man7.org/linux/man-pages/man8/ld.so.8.html
- https://man7.org/linux/man-pages/man2/execve.2.html

### Contents

- [Android SeLinux Policies](#android-selinux-policies)
- [Google PlayStore Policies](#google-play-store-policies)
- [Solutions](#solutions)

---

&nbsp;





## Android SeLinux Policies

The following SeLinux policies exist on Android as part of [`W^X`](https://en.wikipedia.org/wiki/W%5EX) restrictions for executing files in app data directory depending on domain/process context type assigned to the app process and the type assigned to the files under the app data directory.

- [`untrusted_app*`](#untrusted_app-selinux-policy)
- [`priv_app`](#priv_app-selinux-policy)
- [`ephemeral_app`](#ephemeral_app-selinux-policy)
- [`gmscore_app`](#gmscore_app-selinux-policy)
- [Other domains](#se-linux-policy-of-other-domains)

&nbsp;

### `untrusted_app*` SeLinux Policy

Android `>= 10` with the [`0dd738d8`](https://cs.android.com/android/_/android/platform/system/sepolicy/+/0dd738d810532eb41ad8d90520156212ce756648) commit removed the `untrusted_app*` domains assigned to untrusted third party app processes that use `targetSdkVersion` `>= 29` to `execute_no_trans` their app data files assigned the [`app_data_file` file context type](../../os/selinux/context-types.md#app_data_file) and only kept the `execute` permission. Two backward compatibility domains were also added for which both the `execute` and `execute_no_trans` permissions were still allowed, the `untrusted_app_25` domain for apps that use `targetSdkVersion` `<= 25` and `untrusted_app_27` that use `targetSdkVersion` `26-28`. Basically, this means that on Android `>= 10`, if an app uses `targetSdkVersion` `>= 29`, then by default it can only `dlopen()` files but not `exec()`. Only untrusted apps with `targetSdkVersion` `<= 28` can both `dlopen()` and `exec()` files. The [Google PlayStore Policies](#google-play-store-policies) also prohibits executing untrusted code.

> Enforce execve() restrictions for API `> 28`

> untrusted_app: Remove the ability to run `execve()` on files within an application's home directory. Executing code from a writable /home directory is a `W^X` violation (https://en.wikipedia.org/wiki/W%5EX). Additionally, loading code from application home directories violates a security requirement that all executable code mapped into memory must come from signed sources, or be derived from signed sources.

> Note: this change does *not* remove the ability to load executable code through other mechanisms, such as `mmap(PROT_EXEC)` of a file descriptor from the app's home directory. In particular, functionality like `dlopen()` on files in an app's home directory continues to work even after this change.

> `untrusted_app_25` and `untrusted_app_27`: For backwards compatibility, continue to allow these domains to `execve()` files from the application's home directory.

```shell
// system/sepolicy/private/app_neverallows.te

# Block calling execve() on files in an apps home directory.
# This is a W^X violation (loading executable code from a writable
# home directory). For compatibility, allow for targetApi <= 28.
# b/112357170
neverallow {
  all_untrusted_apps
  -untrusted_app_25
  -untrusted_app_27
  -runas_app
} { app_data_file privapp_data_file }:file execute_no_trans;
```

```shell
// system/sepolicy/private/untrusted_app_all.te

# Some apps ship with shared libraries and binaries that they write out
# to their sandbox directory and then execute.
allow untrusted_app_all privapp_data_file:file { r_file_perms execute };
allow untrusted_app_all app_data_file:file     { r_file_perms execute };
auditallow untrusted_app_all app_data_file:file execute;
```

```shell
// system/sepolicy/private/untrusted_app_27.te

# The ability to call exec() on files in the apps home directories
# for targetApi 26, 27, and 28.
allow untrusted_app_27 app_data_file:file execute_no_trans;
auditallow untrusted_app_27 app_data_file:file { execute execute_no_trans };
```

- [`untrusted_app*` process context type](../../os/selinux/context-types.md#untrusted_app)
- https://android-review.googlesource.com/c/platform/system/sepolicy/+/804149
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/4738b93db250175f0915cee2f08ab01aaf8d28f9
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/c47e149a0b8c16339bbebe43e5033dd1174035d3
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/23c9d91b46352bd91cdc58f33d55378e5567dc1c
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/0dd738d810532eb41ad8d90520156212ce756648
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/fb66c6f81b9fae71be381d30b7ebb4a84756df02
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r31:system/sepolicy/private/app_neverallows.te;l=58-67
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r31:system/sepolicy/private/untrusted_app_all.te;l=24-28
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r31:system/sepolicy/private/untrusted_app_27.te;l=22-25
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r31:system/sepolicy/public/global_macros;l=25

#### See also

- [`issuetracker#128554619`](https://issuetracker.google.com/issues/128554619)
- [`termux/termux-app#1072`: No more exec from data folder on targetAPI >= Android Q](https://github.com/termux/termux-app/issues/1072)
- [`termux/termux-app#2155`: Revisit the Android W^X problem](https://github.com/termux/termux-app/issues/2155)

## &nbsp;



### `priv_app` SeLinux Policy

Android `>= 8` removed the permission for `priv_app` domain to `execute_no_trans` an `app_data_file`, and only kept the `execute` permission. In Android `10`, the `priv_app` app data file label was switched from `app_data_file` to `privapp_data_file` and execute permission was kept. This was done to separate policies for `app_data_file` that were assigned to `untrusted_app*` domains. Basically, this means that on Android `>= 8`, `priv_app` by default can only `dlopen()` files but not `exec()`.

- [`priv_app` process context type](../../os/selinux/context-types.md#priv_app)
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/758e6b36784d1a707c8b2813f89f1edc023d59c8
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/8fb4cb8bc20b999d69210f6fc6b552c6e22b8e09
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/23c9d91b46352bd91cdc58f33d55378e5567dc1c
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/5d1755194a841ab727467a30757fd1606cef905b
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/e1ddd741deb7ef393d907dd2bd089e36847f831f
- https://cs.android.com/android/platform/superproject/+/android-7.1.2_r39:system/sepolicy/priv_app.te;l=14-16
- https://cs.android.com/android/platform/superproject/+/android-8.0.0_r1:system/sepolicy/private/priv_app.te;l=20-22
- https://cs.android.com/android/platform/superproject/+/android-10.0.0_r1:system/sepolicy/private/priv_app.te;l=20-29
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r31:system/sepolicy/private/priv_app.te;l=17-26

## &nbsp;


### `ephemeral_app` SeLinux Policy

Android `8.1` allowed the `ephemeral_app` domain for apps that run without installation (like via Google Play Instant) the `execute` permission but not `execute_no_trans` so that they can `dlopen()` files but not `exec()`.

- [`ephemeral_app` process context type](../../os/selinux/context-types.md#ephemeral_app)
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/7650669fe88f4874291c014c41d0235183d79bcf
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/0eb0a16fbdfbb986d4209c86fe0261db35129c01
- https://cs.android.com/android/platform/superproject/+/android-8.1.0_r1:system/sepolicy/private/ephemeral_app.te;l=22-24
- https://cs.android.com/android/platform/superproject/+/android-10.0.0_r1:system/sepolicy/private/ephemeral_app.te;l=22-25

## &nbsp;



### `gmscore_app` SeLinux Policy

Android `11` created the `gmscore_app` for Google Play services (`com.google.android.gms`) and Google Services Framework (`com.google.android.gsf`) to give it `execute` permission but not `execute_no_trans` so that they can `dlopen()` files but not `exec()`.

- [`gmscore_app` process context type](../../os/selinux/context-types.md#gmscore_app)
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/c46a7bc759049445cf1b2d09eab4d5018f690aa8
- https://cs.android.com/android/platform/superproject/+/android-11.0.0_r1:system/sepolicy/private/gmscore_app.te;l=67-76
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/seapp_contexts;l=188-191

## &nbsp;



###  SeLinux Policy of Other Domains

The other domains like `isolated_app`, `system_app` and `platform_app` do not have any `execute` permissions for their app data files.

---

&nbsp;





## Google PlayStore Policies

Executing code downloaded (or compiled) at runtime from sources outside the Google Play Store, like not packed inside the apk is not allowed.

> An app distributed via Google Play may not modify, replace, or update itself using any method other than Google Play's update mechanism. Likewise, an app may not download executable code (such as dex, JAR, .so files) from a source other than Google Play. This restriction does not apply to code that runs in a virtual machine or an interpreter where either provides indirect access to Android APIs (such as JavaScript in a webview or browser).
> Apps or third-party code, like SDKs, with interpreted languages (JavaScript, Python, Lua, etc.) loaded at run time (for example, not packaged with the app) must not allow potential violations of Google Play policies.

- https://support.google.com/googleplay/android-developer/answer/9888379?hl=en

---

&nbsp;




## Solutions

The following solutions exists to workaround or bypass the `W^X` restrictions for executing app data files.

- [APK Native Library](#a-p-k-native-library)
- [System Linker Exec](#system-linker-exec)
- [Patch SeLinux Policies](#patch-se-linux-policies)
- [Other solutions](#other-solutions)

#### APK Native Library

Android allows both `dlopen()` and `exec()` of files that were added as native libraries to the APK during build time and then extracted to APK native library directory ([`ApplicationInfo.nativeLibraryDir`](https://developer.android.com/reference/android/content/pm/ApplicationInfo#nativeLibraryDir)) under the `/data/app/*<package_name>/lib/<arch>` directory during install time. The extraction can be enabled with the [`extractNativeLibs`](https://developer.android.com/guide/topics/manifest/application-element#extractNativeLibs)/[`useLegacyPackaging`](https://developer.android.com/reference/tools/gradle-api/7.1/com/android/build/api/dsl/JniLibsPackagingOptions#uselegacypackaging) options. The parent directory path for the pre-built executable/binary files can be set in the  [`android.sourceSets.main.jniLibs.srcDirs`](https://github.com/agnostic-apollo/TaskerAppFactory/blob/dbf319ff/app/build.gradle#L38) [source sets](https://developer.android.com/build/build-variants#sourcesets) gradle config in the app `build.gradle` file. **The filename of all files must end with `.so`, otherwise they will not be added to the APK.**

The native library files of other apps can also be executed even if their APKs have been signed with a different signing key or their [`sharedUserId`](https://developer.android.com/guide/topics/manifest/manifest-element#uid) is different or isn't set, so if you cannot add all the executable files and dependencies to a single app, additional plugin apps can be created and installed.

The files in the APK native library directory `/data/app/*<package_name>/lib/<arch>` have the `rwxr-xr-x` permission, basically `others` can `read` and `execute` them, but not `write` to them. The app APK files and their extracted native libraries in the `/data/app/*` directories are owned by the `system` user and not by the user assigned to the app at install time. They are assigned the SeLinux [`apk_data_file*` file context type](../../os/selinux/context-types.md#apk_data_file) and SeLinux policies in `app.te` allows any `untrusted_app`, `platform_app` and `priv_app` to `execute` and `execute_no_trans` the native libraries (but not a `system_app` from a `rw` `/data` partition). The app data files under `/data/data` are assigned the different file context type like [`app_data_file` ](../../os/selinux/context-types.md#app_data_file) for which `exec()` is not allowed.

```shell
// system/sepolicy/private/app.te

# Allow apps to read/execute installed binaries
allow appdomain apk_data_file:dir r_dir_perms;
allow appdomain apk_data_file:file rx_file_perms;

# Sensitive app domains are not allowed to execute from /data
# to prevent persistence attacks and ensure all code is executed
# from read-only locations.
neverallow {
  bluetooth
  isolated_app
  nfc
  radio
  shared_relro
  sdk_sandbox
  system_app
} {
  data_file_type
  -apex_art_data_file
  -dalvikcache_data_file
  -system_data_file # shared libs in apks
  -apk_data_file
}:file no_x_file_perms;
```

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/private/app.te;l=412
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/private/app.te;l=482
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/public/file.te;l=326
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/private/file_contexts;l=550
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/public/global_macros;l=33
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/public/global_macros;l=27
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/public/neverallow_macros;l=5
.
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/public/app.te;l=119
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/public/app.te;l=179
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r54:system/sepolicy/private/untrusted_app_27.te;l=18

#### POC

A working (POC) example of this is the [`TaskerAppFactory`](https://github.com/agnostic-apollo/TaskerAppFactory), for which the binaries were compiled with the [`termux-packages` build infrastructure](https://github.com/termux/termux-packages/wiki/Building-packages) and [bootstrap zip file](https://github.com/termux/termux-packages/wiki/For-maintainers#bootstraps) was created for each architecture with `build-bootstrap.sh` and the [`--android10`](https://github.com/termux/termux-packages/blob/c5deccf8e6c0e3f91dc1921a28d0a14354ec5201/scripts/build-bootstraps.sh#L302) flag. The built bootstrap zips are then added under the [`app/src/main/bootstrapZips`](https://github.com/agnostic-apollo/TaskerAppFactory/tree/dbf319ffdb39e99ceb43cdcddc0799a3698be840/app/src/main/bootstrapZips) directory to be read by [`setupBootstraps()` in `app/build.gradle`](https://github.com/agnostic-apollo/TaskerAppFactory/blob/dbf319ff/app/build.gradle#L62-L153) so that binaries and other package files can be added as APK native libraries under the `app/src/main/bootstrapLibs` directory. At runtime, the native libraries are symlinked by [`BootstrapInstaller.java`](https://github.com/agnostic-apollo/TaskerAppFactory/blob/dbf319ff/app/src/main/java/net/dinglisch/android/appfactory/utils/BootstrapInstaller.java) from APK native library directory to under the app data directory under `/data/data/<package_name` ([`Context.getFilesDir()`](https://developer.android.com/reference/android/content/Context#getFilesDir())) to create a `rootfs` so that they can be executed with a proper linux environment and directory structure. The `libfiles.so` file is also added containing the mapping for each native library to its path under `rootfs` at which it should be symlinked as libraries added to the APK exist under a single directory without a nested directory structure and may have same name conflicts. The `libsymlinks.so` file contains a list of additional symlinks that need to be created like `lib/libz.so` to its versioned file `libz.so.1.2.12`. Creating rootfs with symlinks is not necessary for static binaries which have no shared library dependencies, and native library files can be directly executed from the path returned by `ApplicationInfo.nativeLibraryDir`. Termux by default builds shared binaries that use `DT_RUNPATH` and shared libraries must exist under `/data/data/com.termux/files/usr/lib`.

#### Issues

This solution obviously will be a usability nightmare if a lot of packages need to be installed as APKs and each APK installation will show a UI prompt by android package manager, unless using `pm install` with `adb`/`root` to install. To run locally compiled code, it will first need to be packed into an APK file and then installed and then setup in the rootfs before it is usable. It also has issues with standard linux package managers installation and uninstallation design, like `apt` that Termux currently uses by default, like issues related to `conffiles` and `pre`/`post` install scripts and checks in source code or shell scripts that check if a file is a regular file, since that will fail for symlinks. Only symlinking binaries and for rest of the files that do not need execution, either copying original files from native library directories to rootfs or preferably downloading them from repos is also a possibility to reduce installation and failed regular file checks issues. Package managers like `apt` can be patched so that during installation, they signal the app to install the respective APK for a package and create binary file symlinks, while installing the package `deb` file itself to setup rest of the package.

#### Play Core on-demand feature modules

This will not work if using on-demand feature modules that are installed with Play Core APIs (not [`bundletool`](https://developer.android.com/tools/bundletool)), as they are initially installed under `/data/data/<package_name>/files/splitcompat` and made available to the app, but then at a later time installed by the system to the app apk directory under `/data/app`, when app can be killed/restarted. Since `/data/data` files are assigned the [`app_data_file` file context type](../../os/selinux/context-types.md#app_data_file), `exec()` on native libraries extracted there will not be allowed until they are installed under `/data/app` and are assigned the [`apk_data_file*` file context type](../../os/selinux/context-types.md#apk_data_file). There doesn't seem to be a way to directly install to the system with Play Core APIs, but even if that were possible (like `bundletool` does), for every package installed, the Play Core API app would likely require killing the base app, which is obviously terrible UX.

The feature modules are also only provided by Google PlayStore and so will be unusable on devices that don't have PlayStore or can't access Google (like China). Huawei has its own [Dynamic Ability](https://developer.huawei.com/consumer/en/doc/development/AppGallery-connect-Guides/agc-featuredelivery-introduction) API, but even that would not be available for everyone.

>You are not actually installing them, you are effectively requesting the installation via the Play Core API, which emulates the installation until it can install it properly at a later time (usually overnight). This happens because an installation would otherwise require the restart of the application.

- https://github.com/google/bundletool/issues/155#issuecomment-614126504
- https://developer.android.com/guide/playcore/feature-delivery
- https://developer.android.com/guide/playcore/feature-delivery/on-demand
- https://stackoverflow.com/questions/55505739/where-does-android-store-dynamic-module-apks
- https://github.com/google/bundletool/issues/267

&nbsp;



**See [`termux/termux-app/issues/2155`](https://github.com/termux/termux-app/issues/2155#issuecomment-1636732850) for more details.** In the context of Termux, this design is currently called [Termux Android Package Manager (`TAPM`)](https://github.com/termux/termux-app/blob/3b5018b4/termux-shared/src/main/java/com/termux/shared/termux/TermuxBootstrap.java#L133).

**This is the only solution that is compliant with [Google PlayStore Policies](#google-play-store-policies)** as the executable files will be inside the single or multiple plugin APK files, and uploaded to the PlayStore and not downloaded at runtime.

## &nbsp;



### System Linker Exec

The [dynamic linker](https://en.wikipedia.org/wiki/Dynamic_linker) is the part of an operating system that loads and links the shared libraries needed by an executable when it is executed. The kernel is normally responsible for loading both the executable and the dynamic linker. When a [`execve()` system call](https://en.wikipedia.org/wiki/Exec_(system_call)) is made for an [`ELF`](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format#Program_header) executable, the kernel loads the executable file, then reads the path to the dynamic linker from the `PT_INTERP` entry in [`ELF` program header table](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format#Program_header) and then attempts to load and execute this other executable binary for the dynamic linker, which then loads the initial executable image and all the dynamically-linked libraries on which it depends and starts the executable. For binaries built for Android, `PT_INTERP` specifies the path to `/system/bin/linker64` for 64-bit binaries and `/system/bin/linker` for 32-bit binaries. You can check `ELF` file headers with the [`readelf`](https://www.man7.org/linux/man-pages/man1/readelf.1.html) command, like `readelf --program-headers --dynamic /path/to/executable`.

The system provided `linker` at `/system/bin/linker64` on 64-bit Android and `/system/bin/linker` on 32-bit Android can also be passed an absolute path to an executable file on Android `>= 10` for it to execute, even if the executable file itself cannot be executed directly from the app data directory. Note that some 64-devices have 32-bit Android.

An ELF file at `/data/data/com.foo/executable` can be executed with:

```shell
/system/bin/linker64 /data/data/com.foo/executable [args]
```

A script file at `/data/data/com.foo/script.sh` that has the `#!/path/to/interpreter` [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) can be executed with:

```sh
/system/bin/linker64 /path/to/interpreter /data/data/com.foo/script.sh [args]
```

This is possible because when a file is executed with the system linker, the kernel/SeLinux only sees the `linker` binary assigned the `system_linker_exec` file context type being executed by the app process and not the `*app_data_file` being executed by the linker.

Support in Android `linker` to execute files was added in Android `10`, so this method cannot be used on older Android versions, like for some system app domains. For `untrusted_app*` domains, this is not an issue since they can execute files directly on Android `< 10`.

- https://cs.android.com/android/_/android/platform/bionic/+/8f639a40966c630c64166d2657da3ee641303194
- https://cs.android.com/android/_/android/platform/bionic/+/refs/tags/android-10.0.0_r1:linker/linker_main.cpp

SeLinux policy also needs to allow the `execute_no_trans` permission for the app process source context for the `system_linker_exec` file context type.

The permission for `untrusted_app_all` (shared attribute for all `untrusted_app*` domains) was added in android `10.0.0_r1` release tag, and so should be available on all android `>= 10` devices.

```shell
// system/sepolicy/private/untrusted_app_all.te

# Chrome Crashpad uses the the dynamic linker to load native executables
# from an APK (b/112050209, [crbug.com/928422](https://crbug.com/928422))
allow untrusted_app_all system_linker_exec:file execute_no_trans;
```
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/9ea8c0701d162ec40d30b079778723d908e0edca
- https://cs.android.com/android/platform/superproject/+/android-10.0.0_r1:system/sepolicy/public/file.te;l=153
- https://cs.android.com/android/platform/superproject/+/android-10.0.0_r1:system/sepolicy/private/untrusted_app_all.te;l=29

The `priv_app` and `gmscore_app` exemptions were added in android `11.0.0_r38` release tag, and so will not be available on any android `10` devices, only some android `11` devices and all android `>= 12` devices.

```shell
// system/sepolicy/private/priv_app.te

# Chrome Crashpad uses the the dynamic linker to load native executables
# from an APK (b/112050209, [crbug.com/928422](https://crbug.com/928422))
allow priv_app system_linker_exec:file execute_no_trans;
```

```shell
// system/sepolicy/private/gmscore_app.te

# Chrome Crashpad uses the the dynamic linker to load native executables
# from an APK (b/112050209, [crbug.com/928422](https://crbug.com/928422))
allow gmscore_app system_linker_exec:file execute_no_trans;
```

- https://cs.android.com/android/_/android/platform/system/sepolicy/+/970a8fcd2bcd63b3ada0ae5ac25f98d3cd5b994b
- https://cs.android.com/android/platform/superproject/+/android-11.0.0_r38:system/sepolicy/private/priv_app.te;l=28
- https://cs.android.com/android/platform/superproject/+/android-11.0.0_r38:system/sepolicy/private/gmscore_app.te;l=78

**This solution is not compliant with [Google PlayStore Policies](#google-play-store-policies)** as this allows for the executable files to be downloaded from outside the PlayStore at runtime.

#### See also

- [`termux/termux-exec`](https://github.com/termux/termux-exec)
- [`termux/termux-exec#24`](https://github.com/termux/termux-exec/pull/24)

## &nbsp;



### Patch SeLinux Policies

The SeLinux policies can be patched to remove the `W^X` restrictions **if the device is rooted**. This can be done by enabling the `execute` and `execute_no_trans` permission for the [process context type](../../os/selinux/context-types.md#process-context-types) assigned to the app to the [file context type](../../os/selinux/context-types.md#file-context-types) assigned to the files in the app data directory.

**This will allow all apps to execute files that have the same process context type and same file context type for their app data directory as the app for which policies are patches, and so will not just only allow the required app to execute files, which can have security implications, so enable at your own risk.**

The SeLinux policies will need to be patched at **every reboot** of the device by running the commands in a shell or with a [Magisk module](https://topjohnwu.github.io/Magisk/guides.html#magisk-modules) (`sepolicy.rule`), etc. They can also be run in the [`Application.onCreate()`](https://developer.android.com/reference/android/app/Application#onCreate()) of the app with [`Runtime.exec()`](https://developer.android.com/reference/java/lang/Runtime#exec(java.lang.String[],%20java.lang.String[])). **Patching policies requires an API/command to be provided by root manager app, Magisk provides [`magiskpolicy/supolicy`](https://github.com/topjohnwu/Magisk/blob/master/docs/tools.md#magiskpolicy) and SuperSu provides [`supolicy`](https://su.chainfire.eu/#selinux-policies-supolicy) command.**

To get process context type call [`SELinux.getContext()`](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r31:frameworks/base/core/java/android/os/SELinux.java;l=102) or read the `/proc/self/attr/current` or `/proc/<app_pid>/attr/current` file. To get file context type call [`SELinux.getFileContext()`](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r31:frameworks/base/core/java/android/os/SELinux.java;l=80) or run `/system/bin/ls -Z` on the app data directory. To call `SELinux` methods requires bypassing [hidden api reflection restrictions](https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces) and using reflection to call the methods, check `getContext()` and `getFileContext()` methods in [`SELinuxUtils`](https://github.com/termux/termux-app/blob/3b5018b4/termux-shared/src/main/java/com/termux/shared/android/SELinuxUtils.java) class of [`termux-shared`](https://github.com/termux/termux-app/wiki/Termux-Libraries) library.

Following provides commands to patch policies for common app domains.

- [`untrusted_app*`](#untrusted_app-selinux-policy-patch)
- [`priv_app`](#priv_app-selinux-policy-patch)
- [`system_app`](#priv_app-selinux-policy-patch)
- [`platform_app`](#priv_app-selinux-policy-patch)

#### `untrusted_app*` SeLinux Policy Patch

If app uses `targetSdkVersion` `> 28` and is assigned the [`untrusted_app` process context type](../../os/selinux/context-types.md#untrusted_app) or one of the backward compatibility domains `>=` [`untrusted_app_29` process context type](../../os/selinux/context-types.md#untrusted_app_29). In each android release, an additional `untrusted_app_*` domain is normally added unless there are no changes in policies. Replace `untrusted_app` with `untrusted_app_29`, etc if patching for a different `untrusted_app*` process context type.

> `W com.termux: type=1400 audit(0.0:24064): avc: denied { execute_no_trans } for path="/data/data/com.termux/files/usr/bin/login" dev="vdc" ino=123444 scontext=u:r:untrusted_app:s0:c135,c256,c512,c768 tcontext=u:object_r:app_data_file:s0:c135,c256,c512,c768 tclass=file permissive=0`

```shell
su -c 'supolicy --live "allow untrusted_app app_data_file file execute_no_trans"'
```

#### `priv_app` SeLinux Policy Patch

If app is installed as a system app but not signed with platform key and is assigned the [`priv_app` process context type](../../os/selinux/context-types.md#priv_app).

> `W com.termux: type=1400 audit(0.0:186): avc: denied { execute_no_trans } for path="/data/data/com.termux/files/usr/bin/login" dev="mmcblk0p25" ino=393156 scontext=u:r:priv_app:s0:c512,c768 tcontext=u:object_r:privapp_data_file:s0:c512,c768 tclass=file permissive=0`

```shell
su -c 'supolicy --live "allow priv_app privapp_data_file file execute_no_trans"'
```

#### `system_app` SeLinux Policy Patch

If app is installed as a system app and is signed with platform key and uses `sharedUserId=com.android.system` and is assigned the [`system_app` process context type](../../os/selinux/context-types.md#system_app). We also need to allow `execute` permission in addition to `execute_no_trans` since by default it is not allowed, like it is for `untrusted_app` and `priv_app` domains.

> `W/com.termux: type=1400 audit(0.0:1245): avc: denied { execute } for name="login" dev="dm-8" ino=644006 scontext=u:r:system_app:s0 tcontext=u:object_r:system_app_data_file:s0 tclass=file permissive=0`

```shell
su -c 'supolicy --live "allow system_app system_app_data_file file execute" "allow system_app system_app_data_file file execute_no_trans"'
```

#### `platform_app` SeLinux Policy Patch

If app is installed as a system app and is signed with platform key but does not use `sharedUserId=com.android.system` and is assigned the [`platform_app` process context type](../../os/selinux/context-types.md#platform_app). We also need to allow `execute` permission in addition to `execute_no_trans` since by default it is not allowed, like it is for `untrusted_app` and `priv_app` domains.

```shell
su -c 'supolicy --live "allow platform_app app_data_file file execute"  "allow platform_app app_data_file file execute_no_trans"'
```

**This solution is not compliant with [Google PlayStore Policies](#google-play-store-policies)** as this allows for the executable files to be downloaded from outside the PlayStore at runtime and modifying default system SeLinux policies with root may cause problems with approvals too.

- https://github.com/termux/termux-app/issues/2445#issuecomment-985484952
- [4.4.1.3. SELinux Specific Permissions](https://flylib.com/books/en/2.803.1.34/1/) ([libgen](http://libgen.rs/search.php?req=SELinux+by+Example))

## &nbsp;




### Other solutions

There are some other solutions discussed in [`termux/termux-app#1072`](https://github.com/termux/termux-app/issues/1072) and [`termux/termux-app#2155`](https://github.com/termux/termux-app/issues/2155#issuecomment-1008669767).

#### Proot

[`proot`](https://proot-me.github.io) can be used to execute binaries, however it has a performance impact and is [not stable on all devices](https://github.com/termux/proot/issues), especially old devices.

#### `untrusted_dev_app` process context type

Since SeLinux is what restricts execution, the only way to exempt an app from such a restriction would be assigning it a different [process context type](../../os/selinux/context-types.md#process-context-types) that does have the required `execute_no_trans` permission. The elevated process context type should only be assigned if an app has been granted a dangerous runtime or development permission granted with `adb` so that only specific development apps that the users specifically allows are allowed to execute untrusted code. **This is currently not possible and is planned to be requested from Google/Android. Check [`termux/termux-app/issues/2155`](https://github.com/termux/termux-app/issues/2155#issuecomment-1421268867) for more internal details and is the ideal solution for users who want to retain traditional rootfs and package manager design that allows compiling or downloading executable code and are willing to take the responsibility for any potential risks as per the UNIX way.**

> It is not UNIX’s job to stop you from shooting your foot. If you so choose to do so, then it is UNIX’s job to deliver Mr. Bullet to Mr Foot in the most efficient way it knows. - Terry Lambert

---

&nbsp;
