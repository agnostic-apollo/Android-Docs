# Context Types

#### See also

- [Security Context](security-context.md)

### Contents

- [Process Context Types](#process-context-types)
- [File Context Types](#file-context-types)

---

&nbsp;





## Process Context Types

The process context types assigned to apps are defined in `seapp_contexts`.

An app started by Zygote will have the `appdomain` attribute.

- https://cs.android.com/android/platform/superproject/+/bea25463132c3f4bb35816d175bbe8551f11fc9d:system/sepolicy/README.apps.md
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/seapp_contexts
.
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/public/te_macros;l=206-233
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/public/attributes;l=203-207
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/app.te
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/app_neverallows.te

The common process context types are listed below.

- [`untrusted_app`](#untrusted_app)
- [`untrusted_app_32`](#untrusted_app_32)
- [`untrusted_app_30`](#untrusted_app_30)
- [`untrusted_app_29`](#untrusted_app_29)
- [`untrusted_app_27`](#untrusted_app_27)
- [`untrusted_app_25`](#untrusted_app_25)
- [`system_app`](#system_app)
- [`platform_app`](#platform_app)
- [`priv_app`](#priv_app)
- [`isolated_app`](#isolated_app)
- [`ephemeral_app`](#ephemeral_app)
- [`sdk_sandbox`](#sdk_sandbox)
- [`su`](#su)
- [`magisk`](#magisk)
- [`shell`](#shell)

## &nbsp;



### `untrusted_app`

The process context type assigned to untrusted app processes running with `targetSdkVersion` equal to android sdk version or with `targetSdkVersion` `>=` than the max `targetSdkVersion` of the highest `untrusted_app_*` backward compatibility domains.

The `untrusted_app*` domains are assigned to third party apps that are not system apps, like the ones installed from apps stores like Google Play Store, F-Droid, etc or manually installed. This is the default domain for apps, unless a more specific criteria applies. All apps that have the `untrusted_app*` domain, will also have the `untrusted_app_all` attribute.

When an app is using `targetSdkVersion` `<` the android API level of the device, it may instead be assigned an `untrusted_app_x` domain where `x` is a number for `targetSdkVersion` to signify policies of which older target level will engage for the app instead of the latest ones. For instance, an app with `targetSdkVersion` `32` in its manifest will be typed as [`untrusted_app_32`](#untrusted_app_32). Not all `targetSdkVersion` have a specific type, some version are skipped when no differences were introduced (see `public/untrusted_app.te` for more details).

- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/seapp_contexts;l=192
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/public/untrusted_app.te;l=19
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:/system/sepolicy/private/untrusted_app_all.te
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/untrusted_app.te;l=13-14
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/6231b4d9fc98bb42956198e9f54cabde69464339

See also [`app_data_file`](#app_data_file).

## &nbsp;



### `untrusted_app_32`

The process context type assigned to untrusted app processes running with `targetSdkVersion` `32-33`.

- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/seapp_contexts;l=193
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/public/untrusted_app.te;l=22
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/untrusted_app_32.te

## &nbsp;



### `untrusted_app_30`

The process context type assigned to untrusted app processes running with `targetSdkVersion` `30-31`.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=172
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/public/untrusted_app.te;l=22
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/untrusted_app_30.te

## &nbsp;



### `untrusted_app_29`

The process context type assigned to untrusted app processes running with `targetSdkVersion` `= 29`.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=173
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/public/untrusted_app.te;l=25
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/untrusted_app_29.te

## &nbsp;



### `untrusted_app_27`

The process context type assigned to untrusted app processes running with `targetSdkVersion` `26-28`.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=174
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/public/untrusted_app.te;l=28
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/untrusted_app_27.te

## &nbsp;



### `untrusted_app_25`

The process context type assigned to untrusted app processes running with `targetSdkVersion` `<= 25`.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=176
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/public/untrusted_app.te;l=31
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:/system/sepolicy/private/untrusted_app_25.te

## &nbsp;



### `system_app`

The process context type assigned to system app processes that are signed with the platform key and use `sharedUserId=com.android.system`. The `com.android.settings` is an example of an app running with this type.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=146
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/system_app.te

See also [`system_app_data_file`](#system_app_data_file).

## &nbsp;



### `platform_app`

The process context type assigned to platform app processes that are signed with the platform key and do not use `sharedUserId=com.android.system`. These are installed within the system or vendor image. The `com.android.systemui` is an example of an app running with this type.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=159
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/platform_app.te

See also [`app_data_file`](#app_data_file).

## &nbsp;



### `priv_app`

The process context type assigned to privileged app processes that are not signed with the platform key and are shipped as part of the device and installed in one of the
`/{system,vendor,product}/priv-app` directories. The `com.google.android.apps.messaging` is an example of an app running as priv_app. Permissions for these apps need to be explicitly granted, see https://source.android.com/docs/core/permissions/perms-allowlist for more details.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=161
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/priv_app.te

See also [`privapp_data_file`](#privapp_data_file).

## &nbsp;



### `isolated_app`

The process context type assigned to app service processes with `isolatedProcess=true` in their `AndroidManifest.xml`.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=155
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/isolated_app.te
- https://developer.android.com/guide/topics/manifest/service-element#isolated

## &nbsp;



### `ephemeral_app`

The process context type assigned to app processes for apps that are run without installation. These are apps deployed for example via Google Play Instant. These are more constrained than [`untrusted_app`](#untrusted_app) domain.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=160
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/ephemeral_app.te
- https://developer.android.com/topic/google-play-instant

## &nbsp;



### `sdk_sandbox`

The process context type assigned to app processes of SDK runtime apps, installed as part of the Privacy Sandbox project. These are sandboxed to limit their communication channels.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=156
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/sdk_sandbox.te
- https://developer.android.com/design-for-safety/privacy-sandbox
- https://developer.android.com/design-for-safety/privacy-sandbox/sdk-runtime

## &nbsp;



### `gmscore_app`

The process context type assigned to Google Play services (`com.google.android.gms`) and Google Services Framework (`com.google.android.gsf`) apps.

- https://cs.android.com/android/_/android/platform/system/sepolicy/+/c46a7bc759049445cf1b2d09eab4d5018f690aa8
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/4a1630133dc7c98da923aab5c3261432d311766b
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/efc3bdb2558eeb38fe72f8231f13bc2469e892e6
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=167-170
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/gmscore_app.te


## &nbsp;



### `su`

The process context type assigned to `root` (`0`) user processes by AOSP, like for `adb root`. The process context always equals `u:r:su:s0` without any [categories](security-context.md#categories).

- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/su.te
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/core/rootdir/init.usb.rc;l=15
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/init.te;l=32

## &nbsp;



### `magisk`

The process context type assigned to `root` (`0`) user processes by [Magisk](https://github.com/topjohnwu/Magisk), like for `su` commands. The process context always equals `u:r:magisk:s0` without any [categories](security-context.md#categories). **Note that unfortunately Magisk is not part of AOSP, still waiting on John Wu to take over Google from the inside. ;)**

- https://github.com/topjohnwu/Magisk/blob/master/native/src/include/consts.hpp#L36
- https://github.com/topjohnwu/Magisk/blob/master/docs/tools.md#magiskpolicy

## &nbsp;



### `shell`

The process context type assigned to `shell` (`2000`) user, like for `adb shell`. The process context always equals `u:r:shell:s0` without any [categories](security-context.md#categories).

- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/seapp_contexts;l=168
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/shell.te
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/core/rootdir/init.rc
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:packages/modules/adb/daemon/shell_service.cpp;l=379

---

&nbsp;





## File Context Types

The file context types assigned to apps are defined in `file.te`.

- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/public/file.te

The common process context types are listed below.

- [`app_data_file`](#app_data_file)
- [`system_app_data_file`](#system_app_data_file)
- [`privapp_data_file`](#privapp_data_file)
- [`apk_data_file`](#apk_data_file)
- [`shell_data_file`](#shell_data_file)

## &nbsp;



### `app_data_file`

The file context type assigned to app data files under `/data/data/*` and related directories of app processes with [`untrusted_app`](#untrusted_app) and [`platform_app`](#platform_app) domains, and with [`priv_app`](#priv_app) domain on Android `<= 9`.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/public/file.te;l=457-458
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=159
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=171-176
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/public/attributes;l=49

## &nbsp;



### `system_app_data_file`

The file context type assigned to app data files under `/data/data/*` and related directories of app processes with [`system_app`](#system_app) domain.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/public/file.te;l=461-462
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/seapp_contexts;l=146
-

## &nbsp;



### `privapp_data_file`

The file context type assigned to app data files under `/data/data/*` and related directories of app processes with [`priv_app`](#priv_app) domain on Android `>= 10`.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/public/file.te;l=459-460
- https://cs.android.com/android/platform/superproject/+/android-10.0.0_r1:system/sepolicy/private/seapp_contexts;l=158
- https://cs.android.com/android/platform/superproject/+/android-9.0.0_r61:system/sepolicy/private/seapp_contexts;l=115
.
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/8fb4cb8bc20b999d69210f6fc6b552c6e22b8e09
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/23c9d91b46352bd91cdc58f33d55378e5567dc1c
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/5d1755194a841ab727467a30757fd1606cef905b
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/e1ddd741deb7ef393d907dd2bd089e36847f831f

## &nbsp;



### `apk_data_file`

The file context type assigned to app APK files and their extracted native libraries under `/data/app/*<package_name>/` directories of app processes. The `oat` files are assigned the `dalvikcache_data_file` type instead.

- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/public/file.te;l=326-327
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:system/sepolicy/private/file_contexts;l=548
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/2dd4e51d5c2a2dfc0bfdee9303269f5a665f6e35

## &nbsp;



### `shell_data_file`

The file context type assigned to files under `/data/local/*` of processes with [`shell`](#shell) domain.

- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/public/file.te;l=356-357
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r1:system/sepolicy/private/seapp_contexts;l=168
-

## &nbsp;

---

&nbsp;
