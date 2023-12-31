# Security Context

The security context label is in the format `user:role:type:sensitivity[:categories]` containing `4` or `5` components, where on Android [`user`](#user) is always `u`, [`role`](#role) is always `r` for subjects (example: processes) and `object_r` for objects (example: files), and [`sensitivity`](#sensitivity) is always `s0`. The [`categories`](#categories) are optional, but if set, then either `2` or `4` categories are set where each category starts with `c` followed by a number. For non system apps, it will be something like `u:r:untrusted_app:s0:c159,c256,c512,c768`. Check [`categories`](#categories) for more info.

#### See also

- [Context Types](context-types.md)
- https://source.android.com/docs/security/features/selinux/concepts#security_contexts
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:external/selinux/libselinux/src/context.c;l=17
- https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:external/selinux/libselinux/src/android/android_seapp.c;l=686


### Contents

- [Components](#components)
	- [`user`](#user)
	- [`role`](#role)
	- [`type`](#type)
	- [`sensitivity`](#sensitivity)
	- [`categories`](#categories)

---

&nbsp;





## Components

The following components exist in a security context.

&nbsp;



### `user`

SELinux users are not used by android. The only user defined is `u`. When necessary, physical users are represented using the categories field of a security context.

## &nbsp;



### `role`

SELinux roles and Role-Based Access Control (RBAC) are not used by android. Two default roles are defined and used: `r` for subjects (example: processes) and `object_r` for objects (example: files).

## &nbsp;



### `type`

The type (or attribute) of the subject.

## &nbsp;



### `sensitivity`

SELinux sensitivities are not used by android. The default `s0` sensitivity is always set.

## &nbsp;



### `categories`

The categories are in the format `c[0-9]+,c[0-9]+,c[0-9]+,c[0-9]+` and are part of `Multi-Category Security (MCS)` and required to `Isolate the app data from access by another app` and `Isolate the app data from one physical user to another`.

The `untrusted_app_25` should have 2 categories, while `>=` `untrusted_app_27` should have `4` categories if app has `targetSdkVersion` `>= 28`. For example for uid `10159` and user `0`
security context with be set to `u:r:untrusted_app_27:s0:c159,c256,c512,c768` and for uid `1010159` and user `10` it will be set to `u:r:untrusted_app_27:s0:c159,c256,c522,c768`,
note `c512` vs `c522`. This is because the selinux domain `untrusted_app_25` and `untrusted_app_27` using older `targetSdkVersion` use `levelFrom=user` selector in SELinux `seapp_contexts` instead of `levelFrom=all`, which adds two category types.

Both the process and file security context have categories assigned and the process categories must be a superset or equal the file categories to be allowed access. For example for `c160,c256,c512,c768`:

- The first category `c160` is for `appid` for `uid` `10160` and it would get changed to `c161` for `uid` `10161`. The second category `c256` would get incremented to `c257` when `appid` crosses `255` and first category would get reset to `c0` and same will happen when `appid` crosses `511`.
- The third category `c512` is for `userid` `0` (primary user) and it would get changed to `c522` for `userid` `10` (secondary user). The forth category `c768` would get incremented to `c769` when `userid` crosses `255` and third category would get reset to `c0` and same will happen when `userid` crosses `511`.
- A process context with security context `c160,c256,c512,c768` will be able to access files with categories `c160,c256,c512,c768` or files with no categories but not with categories `c161,c256,c512,c768` (different `appid`) or `c160,c256,c522,c768` (different `userid`).
- The `uid` for normal apps with `AID_APP_START >= uid < AID_SDK_SANDBOX_PROCESS_START` is converted to `appid` with `((uid % AID_USER_OFFSET) - AID_APP_START)`.
- The `uid` is assigned to apps with `(userid * AID_USER_OFFSET + AID_APP_START + appid)`.

```
AID_USER_OFFSET 100000 /* offset for uid ranges for each user */
AID_APP_START 10000 /* first app user */
AID_APP_END 19999   /* last app user */
profile user 0: (0 * 100000 + 10000 + 160) -> 10160/u0_a160
profile user 10: (10 * 100000 + 10000 + 160) -> 1010160/u10_a160
profile user 256: (256 * 100000 + 10000 + 160) -> 25610160/u256_a160
```

```java
public static String getCategories(int uid) {
    int userid = (uid / 100000 /* AID_USER_OFFSET */);
    int appid = ((uid % 100000 /* AID_USER_OFFSET */) - 10000 /* AID_APP_START */);
    return "c" + (appid & 0xff) +
            ",c" + (256 + (appid>>8 & 0xff)) +
            ",c" + (512 + (userid & 0xff)) +
            ",c" + (768 + (userid>>8 & 0xff));
}
```

```java
# (uid, userid) = (categories)

(10000,0) = (c0,c256,c512,c768)
(10088,0) = (c88,c256,c512,c768)
(10099,0) = (c99,c256,c512,c768)
(10100,0) = (c100,c256,c512,c768)
(10160,0) = (c160,c256,c512,c768)
(10212,0) = (c212,c256,c512,c768)
(10255,0) = (c255,c256,c512,c768)
(10256,0) = (c0,c257,c512,c768)
(10511,0) = (c255,c257,c512,c768)
(10512,0) = (c0,c258,c512,c768)
(10593,0) = (c81,c258,c512,c768)
(10600,0) = (c88,c258,c512,c768)
(10999,0) = (c231,c259,c512,c768)
(11000,0) = (c232,c259,c512,c768)
(1010000,10) = (c0,c256,c522,c768)
(1010088,10) = (c88,c256,c522,c768)
(1010099,10) = (c99,c256,c522,c768)
(1010100,10) = (c100,c256,c522,c768)
(1010160,10) = (c160,c256,c522,c768)
(1010212,10) = (c212,c256,c522,c768)
(1010255,10) = (c255,c256,c522,c768)
(1010256,10) = (c0,c257,c522,c768)
(1010511,10) = (c255,c257,c522,c768)
(1010512,10) = (c0,c258,c522,c768)
(1010593,10) = (c81,c258,c522,c768)
(1010600,10) = (c88,c258,c522,c768)
(1010999,10) = (c231,c259,c522,c768)
(1011000,10) = (c232,c259,c522,c768)
(25610160,256) = (c160,c256,c512,c769)
(25610255,256) = (c255,c256,c512,c769)
(25610256,256) = (c0,c257,c512,c769)
(25610511,256) = (c255,c257,c512,c769)
(25610512,256) = (c0,c258,c512,c769)
(25610600,256) = (c88,c258,c512,c769)
```

- https://github.com/SELinuxProject/selinux-notebook/blob/main/src/mls_mcs.md#multi-level-and-multi-category-security
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux/using-multi-level-security-mls_using-selinux
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux/assembly_using-multi-category-security-mcs-for-data-confidentiality_using-selinux
- https://selinuxproject.org/page/NB_SEforAndroid_2#Computing_a_Context
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:cts/tests/tests/selinux/selinuxTargetSdk25/src/android/security/SELinuxTargetSdkTest.java;l=38
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:cts/tests/tests/selinux/selinuxTargetSdk28/src/android/security/SELinuxTargetSdkTest.java;l=49
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:cts/tests/tests/selinux/selinuxTargetSdkCurrent/src/android/security/SELinuxTargetSdkTest.java;l=56
- https://cs.android.com/android/platform/superproject/+/android-13.0.0_r18:cts/tests/tests/selinux/common/src/android/security/SELinuxTargetSdkTestBase.java;l=134
- https://cs.android.com/android/platform/superproject/+/066bed72:external/selinux/libselinux/src/android/android_seapp.c;l=743
- https://cs.android.com/android/platform/superproject/+/066bed72:external/selinux/libselinux/src/android/android_seapp.c;l=660
- https://cs.android.com/android/platform/superproject/+/066bed72:system/core/libcutils/include/private/android_filesystem_config.h;l=197
- https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:bionic/libc/bionic/grp_pwd.cpp;l=274
- https://cs.android.com/android/_/android/platform/system/sepolicy/+/6231b4d9fc98bb42956198e9f54cabde69464339

---

&nbsp;
