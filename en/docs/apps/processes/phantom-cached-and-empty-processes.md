# Phantom, Cached And Empty Processes

This was originally posted in a comment in [`termux/termux-app` issue `#2366`](https://github.com/termux/termux-app/issues/2366#issuecomment-961305970) references info in relation to it, its comments and the original issue creator (OP/@V-no-A) and then posted at https://gist.github.com/agnostic-apollo/dc7e47991c512755ff26bd2d31e72ca8.

An additional issue was opened at google's `issuetracker` at https://issuetracker.google.com/issues/205156966 since the phantom process and excessive cpu usage killing changes were silently introduced in Android `12` breaking lot of multiple apps, including **core functionality** of [Termux](https://github.com/termux/termux-app) and [Tasker](https://tasker.joaoapps.com) and other automation apps.

Google was asked for an option to disable such killing and a **huge thanks to Jing Ji** for adding the `settings_enable_monitor_phantom_procs` settings flag with [`09dcdad`](https://cs.android.com/android/_/android/platform/frameworks/base/+/09dcdad5ebc159861920f090e07da60fac71ac0a) to allow disabling both phantom process and excessive cpu usage, which is available since Android `12L beta 3`. Check [comment 27](https://issuetracker.google.com/issues/205156966#comment27) and [comment 54](https://issuetracker.google.com/issues/205156966#comment54). Check [How to disable the phantom processes killing?](#how-to-disable-the-phantom-processes-killing) for details on how to disable the killing on different android versions.





## Contents
- [Phantom Processes](#phantom-processes)
  - [What are phantom process?](#what-are-phantom-processes)
  - [Which apps will this affect?](#which-apps-will-this-affect)
  - [Phantom process bookkeeping](#phantom-process-bookkeeping)
  - [How are phantom processes killed?](#how-are-phantom-processes-killed)
    - [Excessive CPU](#excessive-cpu)
    - [Max Phantom Processes](#max-phantom-processes)
  - [How to check max phantom processes that are allowed to run?](#how-to-check-max-phantom-processes-that-are-allowed-to-run)
  - [How to check phantom processes running that are being monitored by `PhantomProcessList`?](#how-to-check-phantom-processes-running-that-are-being-monitored-by-phantomprocesslist)
  - [How to detect phantom processes that were killed?](#how-to-detect-phantom-processes-that-were-killed)
  - [How phantom process killing gets scheduled?](#how-phantom-process-killing-gets-scheduled)
  - [How to disable the phantom processes killing?](#how-to-disable-the-phantom-processes-killing)
    - [Commands to disable phantom process killing and TLDR](#commands-to-disable-phantom-process-killing-and-tldr)
    - [Re-enable device config sync](#re-enable-device-config-sync)
    - [Check if device config sync is currently disabled](#check-if-device-config-sync-is-currently-disabled)
    - [Check device config sync mode stored in settings.](#check-device-config-sync-mode-stored-in-settings)
  - [Why device config may get reset?](#why-device-config-may-get-reset)
  - [What are the risks with disabling phantom process killing?](#what-are-the-risks-with-disabling-phantom-process-killing)
  - [How to manually trigger phantom process killing in Termux?](#how-to-manually-trigger-phantom-process-killing-in-termux)
  - [McAfee app gone crazy](#mcafee-app-gone-crazy)
  - [The "unfair" advantage for foreground oriented apps](#the-unfair-advantage-for-foreground-oriented-apps)
  - [Buggy and malicious apps](#buggy-and-malicious-apps)
  - [Improvements in heuristics and exemptions for apps](#improvements-in-heuristics-and-exemptions-for-apps)
- [Cached And Empty Processes](#cached-and-empty-processes)
  - [Cache Settings](#cache-settings)
  - [Why max limit for KILLING?](#why-max-limit-for-killing)
  - [How are cached and empty processes killed?](#how-are-cached-and-empty-processes-killed)
  - [How to check cache settings?](#how-to-check-cache-settings)
  - [What are default cache settings?](#what-are-default-cache-settings)
    - [ASOP](#ASOP)
    - [Pixel Devices](#Pixel-Devices)
  - [How to increase amount of cached and empty processes kept?](#how-to-increase-amount-of-cached-and-empty-processes-kept)
- [`device_config` command](#device_config-command)

---

&nbsp;





# Phantom Processes

Android 12 via [`15755084`](https://cs.android.com/android/_/android/platform/frameworks/base/+/157550849f0430181fa53c8e1b63112c59c6937b) and updated via [`5706277f`](https://cs.android.com/android/_/android/platform/frameworks/base/+/5706277f2d43d5be20a5510d1fb87a53c901ed52) has added the mechanism to monitor forked child processes started by apps and kills them if more than the default `32` are found running.

The app or phantom processes may **also** be killed if they use excessive CPU, which was done previously too, but [only for app process](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r48:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=17669).

>Track the child processes that are forked by app processes

>Apps could use Runtime.exec() to spawn child process and framework
will have no idea about its lifecycle. Now track those processes
whenever we find them - currently during the cpu stats sampling
they could be spotted. If it's consuming too much CPU while its
parent app process are also in the background, kill it.

>By default we allow up to 32 such processes; the process with the
worst oom adj score of their parents will be killed if there are
too many of them.

Check `PhantomProcessList` commit history [here](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;bpv=1).

This change is neither listed in android `12` [changes list](https://developer.android.com/about/versions/12/summary) nor [behavior changes](https://developer.android.com/about/versions/12/behavior-changes-all) and was done silently.

They were named "Phantom" because they were designed to give android devs nightmares when then saw them (allegedly of course).


---

&nbsp;



## What are phantom processes?

So now what are phantom processes from the perspective of android. It's any process that has been forked from the main `app` process and is now either a child of the `app` process or of `init` process. This can be done in the following ways.



### `Runtime.exec()`

The [`Runtime.exec()`](https://developer.android.com/reference/java/lang/Runtime#exec(java.lang.String[],%20java.lang.String[])) is the `Java` API that can be used to `fork` a child processes. The parent of the child process will be the `app` process itself.

Termux uses this for running background [`TermuxTasks`](https://github.com/termux/termux-app/blob/v0.117/termux-shared/src/main/java/com/termux/shared/shell/TermuxTask.java#L93) (in `termux-app` `v0.109-0.118.0`) and are managed by the foreground [`TermuxService`](https://github.com/termux/termux-app/blob/v0.117/app/src/main/java/com/termux/app/TermuxService.java#L86). These background tasks can be sent via the [`RUN_COMMAND Intent`](https://github.com/termux/termux-app/wiki/RUN_COMMAND-Intent) and by termux plugins like [`termux-boot`](https://github.com/termux/termux-boot), [`termux-tasker`](https://github.com/termux/termux-tasker) and [`termux-widget`](https://github.com/termux/termux-widget). They show as `<n> tasks` in the `Termux` notification.



[`Runtime.exec()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:libcore/ojluni/src/main/java/java/lang/Runtime.java;l=694) -> [`ProcessBuilder.start()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:libcore/ojluni/src/main/java/java/lang/ProcessBuilder.java;l=1029) -> [`ProcessImpl.start()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:libcore/ojluni/src/main/java/java/lang/ProcessImpl.java;l=137) -> [`UNIXProcess()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:libcore/ojluni/src/main/java/java/lang/UNIXProcess.java;l=133) -> [`UNIXProcess_md.forkAndExec()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:libcore/ojluni/src/main/native/UNIXProcess_md.c;l=926) -> [`UNIXProcess_md.startChild()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:libcore/ojluni/src/main/native/UNIXProcess_md.c;l=847) ([note on forking](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:libcore/ojluni/src/main/native/UNIXProcess_md.c;l=68) for [`clone()`](https://manpages.debian.org/testing/manpages-dev/clone.2.en.html), [`vfork()`](https://manpages.debian.org/testing/manpages-dev/vfork.2.en.html) and [`fork()`](https://manpages.debian.org/testing/manpages-dev/fork.2.en.html) usage) -> [`UNIXProcess_md.childProcess()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:libcore/ojluni/src/main/native/UNIXProcess_md.c;l=782) -> [`UNIXProcess_md.JDK_execvpe()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:libcore/ojluni/src/main/native/UNIXProcess_md.c;l=603) -> [`exec.execvpe()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r32:bionic/libc/bionic/exec.cpp;l=119)


### `execvp`

The [`execvp`](https://manpages.debian.org/stretch/manpages-dev/execvp.3.en.html) is part of the native `exec()` family of functions that replaces the current process image with a new process image. This can be called by the child process that has been spawned from the `app` process after it called [`fork()`](https://manpages.debian.org/stretch/manpages-dev/fork.2.en.html). These functions can be called in `JNI` in native `c/c++` code.

Termux uses this to start the foreground [`TermuxSessions`](https://github.com/termux/termux-app/blob/v0.117/termux-shared/src/main/java/com/termux/shared/shell/TermuxSession.java#L131) and are managed by the foreground [`TermuxService`](https://github.com/termux/termux-app/blob/v0.117/app/src/main/java/com/termux/app/TermuxService.java#L78). The `TermuxSession` creates a `TerminalSession` that calls [`create_subprocess`](https://github.com/termux/termux-app/blob/v0.117/terminal-emulator/src/main/java/com/termux/terminal/TerminalSession.java#L127) via `JNI` defined in [`termux.c`](https://github.com/termux/termux-app/blob/v0.117/terminal-emulator/src/main/jni/termux.c#L63), which then calls [`fork()`](https://github.com/termux/termux-app/blob/v0.117/terminal-emulator/src/main/jni/termux.c#L63) and the child calls [`execvp()`](https://github.com/termux/termux-app/blob/v0.117/terminal-emulator/src/main/jni/termux.c#L106). By default, if no custom command is passed to run in the `TerminalSession`, the `/data/data/com.termux/files/usr/bin/login` script is called, which runs [`exec "$SHELL"`](https://github.com/termux/termux-packages/blob/master/packages/termux-tools/login#L35) to replace itself with the `login` shell defined, which defaults to `/data/data/com.termux/files/usr/bin/bash`.



### `daemon`

The [`daemon()`](https://manpages.debian.org/bullseye/manpages-dev/daemon.3.en.html) function is for programs wishing to detach themselves from the controlling terminal and run in the background as [system daemons](https://manpages.debian.org/bullseye/systemd/daemon.7.en.html). The child process forked from the parent process basically detaches itself from the parent so that it is no longer its parent process (`ppid`) and is inherited by the `init` (`pid` `1`) process.

Termux provides commands like `sshd` and `crond` to start daemon processes. If `sshd` command is run, the `ps` output will show `sshd` to have `init` (`pid` `1`) as the `ppid`, instead of `pid` of its original parent `bash`. These have a higher chance of getting killed since there are no longer attached to the `app` process. You can optionally [not daemonize](https://github.com/termux/termux-app/issues/2015#issuecomment-860492160) `sshd` and just run it in the background and it will then still be tied to `app` process and less likely to get killed, like with `sshd -D`.


---

&nbsp;





## Which apps will this affect?

This will majorly affect all apps that fork child processes from their app process, like with `Runtime.exec()` or in `JNI`.  

For Termux app, this killing will affect all commands runs in the `shell`, which is basically everything the app does and other plugins are designed around.

For Tasker, this will affect `Run Shell`, `Adb Wifi` and some other actions that run `shell` commands internally, and also the `Logcat Entry` event profile.

Expect such processes to be killed at any time if using Android `12`.


---

&nbsp;





### Phantom process bookkeeping

`3` commands were started below by `Termux` and `dumpsys activity processes` output shows all `3` being bookkeeped and open for killing by `PhantomProcessList`. You can trigger an update for bookkeeping/killing by connecting or disconnecting the charger, as per `Battery power changes` mentioned in [`How phantom process killing gets scheduled?`](#How-phantom-process-killing-gets-scheduled) section below.

1. The `bash` process was started as the terminal `login` shell with a call to `execvp` and is shown with the `com.termux` (`pid` `3451`) as its `ppid`.
2. The `top` process was started via `RUN_COMMAND Intent` with a call to `Runtime.exec()` and is shown with the `com.termux` (`pid` `3451`) as its `ppid`.
3. The `sshd` process was started as a `daemon` and is shown with the `int` (`pid` `1`) as its `ppid`.


```
$ sshd

$ am startservice --user 0 -n com.termux/com.termux.app.RunCommandService \
-a com.termux.RUN_COMMAND \
--es com.termux.RUN_COMMAND_PATH '/data/data/com.termux/files/usr/bin/top' \
--esa com.termux.RUN_COMMAND_ARGUMENTS '-n,5' \
--ez com.termux.RUN_COMMAND_BACKGROUND 'true'

~ $ ps -efww
UID        PID  PPID  C STIME TTY          TIME CMD
u0_a147   3451   339  2  1970 ?        00:00:05 com.termux
u0_a147   3542  3451  0  1970 pts/0    00:00:00 /data/data/com.termux/files/usr/bin/bash -l
u0_a147   6206     1  0  1970 ?        00:00:00 sshd
u0_a147   6704  3451  0  1970 ?        00:00:00 /system/bin/top -n
u0_a147   6207  3542  2  1970 pts/0    00:00:00 ps -efww
```

```
$ adb shell "/system/bin/dumpsys activity processes -a"

  All Active App Child Processes:
    proc #0: PhantomProcessRecord {393544a 3542:3451:bash/u0a147}
      user #0 uid=10147 pid=3542 ppid=3451 knownSince=-10m19s315ms killed=false
      lastCpuTime=100 timeUsed=+40ms oom adj=0 seq=11
    proc #1: PhantomProcessRecord {859f776 6206:3451:sshd/u0a147}
      user #0 uid=10147 pid=6206 ppid=3451 knownSince=-6m34s401ms killed=false
      lastCpuTime=0 oom adj=0 seq=11
    proc #2: PhantomProcessRecord {4c1ac7f 6704:3451:top/u0a147}
      user #0 uid=10147 pid=6704 ppid=3451 knownSince=-1s600ms killed=false
      lastCpuTime=0 oom adj=0 seq=11
```


---

&nbsp;





## How are phantom processes killed?

### Excessive CPU

The [`ActivityManagerService.updateAppProcessCpuTimeLPr()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=14274) and [`ActivityManagerService.updatePhantomProcessCpuTimeLPr()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=14302) handles the killing of app and phantom processes which use excessive CPU respectively. The `logcat` will have `Killing...` entries with `excessive cpu` reason if a process is killed.

Both functions are called by [`ActivityManagerService.checkExcessivePowerUsage()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=14237). The checking of if a process is using excessive CPU is done by [`ActivityManagerService.checkExcessivePowerUsageLPr()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=14327). This is [done](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=1704) every [`POWER_CHECK_INTERVAL`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=304), which defaults to [`5*60*1000(5 mins)`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=127).

I am not currently aware of how app CPU usage works when running in an emulator. With the charging off in emulator settings, I tried running more than `32` `sha256sum` processes for `> 5mins` and my host CPU usage was at `100%` (`96.0°C`) but processes kept running and I didn't see any log entries and neither did battery stats service show "App using battery" notification.



### Max Phantom Processes

The [`PhantomProcessList.trimPhantomProcessesIfNecessary()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;l=416) handles the killing of phantom processes greater than [`max_phantom_processes`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=172), which defaults to [`DEFAULT_MAX_PHANTOM_PROCESSES = 32`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=150). The limit applies to **all apps combined** and not `32` processes per app.

The function prioritizes killing of older processes if two processes have the same oom adj for their **app process** (not their own oom adj). If the app processes of the two processes have different oom adj, then process whose app process has higher oom adj will get killed first. The `logcat` will have `Killing...` entries with `Trimming phantom processes` reason if a process is killed.

The `oom_score_adj` is the score assigned to each process by android/linux kernel that is used in deciding which process to kill under low memory conditions. You can read the oom odj value of a process with its `pid` by running the command `cat /proc/<pid>/oom_score_adj`. The `pid` of all processes running can be found by running the `ps -efww` command. This oom adj value is often being updated and the lower the value, the lesser the chance of it getting killed when low memory killer (`LKM`) gets called, since processes with higher oom adj are killed first. For more info on `LKM` and `OOM`, check [here](https://forum.xda-developers.com/showthread.php?t=2751559), [here](https://lwn.net/Articles/317814/), [here](https://github.com/mcgill-cpslab/mobile-research/wiki/How-Android-manages-background-processes%3F) and [here](https://www.linuxjournal.com/content/android-low-memory-killer-or-out). You can read the [android docs for `OomAdjuster`](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/am/OomAdjuster.md) for more info. Oom adj values are based on [`ProcessList.*_ADJ`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=248).

However, phantom processes are not killed by `LKM`, but by `PhantomProcessList` if more than default `32` processes are running, even if memory is not low.

The phantom process records first get copied to a temp array, then sorted in the order of process that should get killed first being at the end of the array, and then the processes getting killed in reverse order from the end until `<= MAX_PHANTOM_PROCESSES` processes remain.

The [`mPid`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessRecord.java;l=56) is that of the app process, not the "parent process" of the phantom process itself, as [set during `PhantomProcessRecord` creation](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;l=333). Also note that the processes listed under `All Active App Child Processes` in the dump will have their own `oom adj=` shown and not their parent app process's and they are often different as per tests. Value would be based on [`/proc/<pid>/oom_score_adj`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessRecord.java;l=150).

If the phantom processes belong to the same app or to two apps that have the same oom adj for their app process, then the older phantom processes will get killed first. This will occur when the `ra.mState.getCurAdj() != rb.mState.getCurAdj()` condition fails. The [`PhantomProcessRecord.mKnownSince = SystemClock.elapsedRealtime()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessRecord.java;l=82) (time since the system was booted) is used for the comparison. The `return a.mKnownSince < b.mKnownSince ? 1 : -1;` line will put older processes at the end of the list. Then in the `for` loop afterwards, killing is done in reverse order of `mTempPhantomProcesses`, which will result in killing of older ones first. That is why in the [comment above](https://github.com/termux/termux-app/issues/2366#issuecomment-955207975), the oldest `9` processes got killed, of which `bash` was the first, since all phantom processes would have had the same oom adj `r*.mState.getCurAdj()` value, i.e of the Termux app.

If the phantom processes belong to different apps, then if **at the time of trimming**, one app process had lower oom odj, like if an app was brought to the foreground and is visible, or last foreground app, or foreground service app (like music/termux), then phantom processes of other apps will get killed first and then in the reverse order mentioned.

```java
/**
 * Clamp the number of phantom processes to
 * {@link ActivityManagerConstants#MAX_PHANTOM_PROCESSE}, kills those surpluses in the
 * order of the oom adjs of their parent process.
 */
void trimPhantomProcessesIfNecessary() {
    synchronized (mService.mProcLock) {
        synchronized (mLock) {
            mTrimPhantomProcessScheduled = false;
            if (mService.mConstants.MAX_PHANTOM_PROCESSES < mPhantomProcesses.size()) {
                for (int i = mPhantomProcesses.size() - 1; i >= 0; i--) {
                    mTempPhantomProcesses.add(mPhantomProcesses.valueAt(i));
                }
                synchronized (mService.mPidsSelfLocked) {
                    Collections.sort(mTempPhantomProcesses, (a, b) -> {
                        final ProcessRecord ra = mService.mPidsSelfLocked.get(a.mPpid);
                        if (ra == null) {
                            // parent is gone, this process should have been killed too
                            return 1;
                        }
                        final ProcessRecord rb = mService.mPidsSelfLocked.get(b.mPpid);
                        if (rb == null) {
                            // parent is gone, this process should have been killed too
                            return -1;
                        }
                        if (ra.mState.getCurAdj() != rb.mState.getCurAdj()) {
                            return ra.mState.getCurAdj() - rb.mState.getCurAdj();
                        }
                        if (a.mKnownSince != b.mKnownSince) {
                            // In case of identical oom adj, younger one first
                            return a.mKnownSince < b.mKnownSince ? 1 : -1;
                        }
                        return 0;
                    });
                }
                for (int i = mTempPhantomProcesses.size() - 1;
                        i >= mService.mConstants.MAX_PHANTOM_PROCESSES; i--) {
                    final PhantomProcessRecord proc = mTempPhantomProcesses.get(i);
                    proc.killLocked("Trimming phantom processes", true);
                }
                mTempPhantomProcesses.clear();
            }
        }
    }
}

```


---

&nbsp;





## How to check max phantom processes that are allowed to run?

#### Check current value of `max_phantom_processes` being used by `ActivityManagerConstants`.

The value returned by the command would be the one being used by `PhantomProcessList` and should normally be the same as the one stored in device config once its loaded.

  - `root`: `su -c "/system/bin/dumpsys activity settings | grep max_phantom_processes"`
  - `adb`: `adb shell "/system/bin/dumpsys activity settings | grep max_phantom_processes"`


```
$ adb shell "/system/bin/dumpsys activity settings | grep max_phantom_processes"
  max_phantom_processes=32
```



#### Check `max_phantom_processes` value stored in device config.

The value returned by the command will be `null` by default if not set with `device_config put` command. More info on `device_config` in [`Disabling the phantom processes killing with adb or root`](Disabling-the-phantom-processes-killing-with-adb-or-root) section.

  - `root`: `su -c "/system/bin/device_config get activity_manager max_phantom_processes"`
  - `adb`: `adb shell "/system/bin/device_config get activity_manager max_phantom_processes"`

```
$ adb shell "/system/bin/device_config get activity_manager max_phantom_processes"
null
```


---

&nbsp;





## How to check phantom processes running that are being monitored by `PhantomProcessList`?

The `dumpsys activity processes` [dump](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=9358) shows the monitored phantom processes under the [`All Active App Child Processes` and `All Zombie App Child Processes`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;l=559) sections. The sections won't be shown if no processes are being monitored. Processes might be running, but still not show since they may not have yet have been detected by ` PhantomProcessList`, but you can trigger an update by connecting or disconnecting the charger, or with other triggers mentioned in [`How phantom process killing gets scheduled?`](#How-phantom-process-killing-gets-scheduled) section below. Check [here](https://stackoverflow.com/questions/20688982/zombie-process-vs-orphan-process) for info on zombie processes.

The `-a` flag minimizes the output. Don't pass on Android `13`.

  - `root`: `su -c "/system/bin/dumpsys activity processes -a"`
  - `adb`:  `adb shell "/system/bin/dumpsys activity processes -a"`

```
$ adb shell "/system/bin/dumpsys activity processes -a"
...
All Active App Child Processes:
    proc #0: PhantomProcessRecord {747692e 7216:6914:bash/u0a147}
      user #0 uid=10147 pid=7216 ppid=6914 knownSince=-29m34s22ms killed=false
      lastCpuTime=20 timeUsed=+30ms oom adj=0 seq=16

```

The `Zombie` entries may be something like following.

```
All Zombie App Child Processes:
  proc #0: PhantomProcessRecord {244efbb 11686:6914:sha256sum/u0a147}
    user #0 uid=10147 pid=11686 ppid=6914 knownSince=-10m12s335ms killed=true
    lastCpuTime=0 oom adj=0 seq=45
```
&nbsp;



#### Run in termux without `root` or `adb`

If you want to run the command as the termux user in the app, then first grant one time permissions over `adb`. You will require [github actions debug build](https://github.com/termux/termux-app#github), or wait for `v0.118` to be released. `PACKAGE_USAGE_STATS` was recently requested via 865f29d4, `DUMP` was available in `v0.109`.

```
adb shell pm grant com.termux android.permission.PACKAGE_USAGE_STATS
adb shell pm grant com.termux android.permission.DUMP
```
then run
```
/system/bin/dumpsys activity processes -a
```


---

&nbsp;





## How to detect phantom processes that were killed?

To get which processes are currently running in your app, run `ps -efww` and you should be able to get their `pid` and full `commandline`. Note down the `pid` values for later use.

When you detect the killing, you can run the following commands to dump the entire `logcat` buffer.

1. Dump to sdcard if running on android.

  - `root`: `su -c "/system/bin/logcat -d > /sdcard/logcat.txt"`
  - `adb`:  `adb shell "/system/bin/logcat -d > /sdcard/logcat.txt"`

2. Dump to current directory if running on pc.
  - `adb`:  `adb logcat -d > logcat.txt`

You can also grant termux the one time `READ_LOGS` permission with `adb shell pm grant com.termux android.permission.READ_LOGS`, so that when you run `logcat` command without `adb` or `root`, you get entries for all android components and apps and not just that of the termux app and its plugins.

You should get the following entries in `logcat`. You can search the `logcat` for `Killing`, `killed`, `Trimming phantom processes` and `excessive cpu` entries to quickly find them. Since `logcat` dumps are huge with thousands of lines, you need to use an app that can handle such large files and not crash. You can use [QuickEdit](https://play.google.com/store/apps/details?id=com.rhmsoft.edit) or the [QuickEdit Pro](https://play.google.com/store/apps/details?id=com.rhmsoft.edit.pro) app if on android or [SublimeText](https://www.sublimetext.com/download) if on a linux distro/windows or [Notepad++](https://notepad-plus-plus.org/) if on windows.

```
~ $ ps -efww
UID        PID  PPID  C STIME TTY          TIME CMD
u0_a147   5641   340  5  1970 ?        00:00:02 com.termux
u0_a147   5683  5641  0  1970 pts/0    00:00:00 /data/data/com.termux/files/usr/bin/bash
u0_a147   5728  5683  1  1970 pts/0    00:00:00 ps -ef
```

Note that `bash` was started with `pid` `5683` and `logcat` logs the killing.

```
I ActivityManager: Killing PhantomProcessRecord {9ccd0ab 5683:5641:bash/u0a147}: Trimming phantom processes
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5683 in 0ms
I ActivityManager: Process PhantomProcessRecord {9ccd0ab 5683:5641:bash/u0a147} died
```

The `Killing PhantomProcessRecord...` entry is generated by [`PhantomProcessRecord.killLocked()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessRecord.java;l=114), with the reason `Trimming phantom processes` passed by [`PhantomProcessList.trimPhantomProcessesIfNecessary()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;l=454).


---

&nbsp;





## How phantom process killing gets scheduled?

The *culling* of phantom processes gets scheduled via [`PhantomProcessList.scheduleTrimPhantomProcessesLocked()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;l=409) which gets called when a new `PhantomProcessRecord` is found/created in [`PhantomProcessList.getOrCreatePhantomProcessIfNeededLocked()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;l=350), which gets called by [`PhantomProcessList.updateProcessCpuStatesLocked()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;l=533), which is called by [`AppProfiler.updateCpuStatsNow()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/AppProfiler.java;l=1834). The `updateCpuStatsNow()` is called from different places, like when:

- [`Reporting memory usage`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/AppProfiler.java;l=1324)
- [`Wakelock changes`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/BatteryExternalStatsWorker.java;l=314)
- [`Battery power changes`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=2528)
- [`Excessive CPU usage`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=14238)
- [`Battery stats being written to file`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/AppProfiler.java;l=127)
- [`Memory usage (pss) collection of all processes being scheduled`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/AppProfiler.java;l=894) and [getting triggered](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/AppProfiler.java;l=426).

The `pss` collection of all processes is called when various oom adjustments are made by [`OomAdjuster`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java;l=978) and may run every [`DEFAULT_FULL_PSS_LOWERED_INTERVAL(5 mins)`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=126) if under [low memory conditions](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/AppProfiler.java;l=998).

So based on `battery power changes`, one can trigger the *culling* by connecting or disconnecting the charger as well. So termux will get killed if more than `32` phantom processes are running by any app, including termux, depending on oom priority of course.


---

&nbsp;





## How to disable the phantom processes killing?

### Android 12L, 13 and higher

As per `settings_enable_monitor_phantom_procs` settings flag added in [`09dcdad`](https://cs.android.com/android/_/android/platform/frameworks/base/+/09dcdad5ebc159861920f090e07da60fac71ac0a), the [`PhantomProcessList.trimPhantomProcessesIfNecessary()`](https://cs.android.com/android/platform/superproject/+/android-12.1.0_r5:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;l=424) does not kill any **phantom processes** and [`ActivityManagerService.checkExcessivePowerUsage()`](https://cs.android.com/android/platform/superproject/+/android-12.1.0_r5:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=14281) does not kill any **processes using excessive cpu** if [`FeatureFlagUtils.SETTINGS_ENABLE_MONITOR_PHANTOM_PROCS`](https://cs.android.com/android/platform/superproject/+/android-12.1.0_r5:frameworks/base/core/java/android/util/FeatureFlagUtils.java;l=58) is disabled, i.e its value is `false`.

Check [Commands to disable phantom process killing and TLDR](#commands-to-disable-phantom-process-killing-and-tldr) section below for the command to disable the flag.

&nbsp;



### Android 12

On Android 12, **no setting exists** that can be changed with `adb` or `root` to disable killing of **processes using excessive cpu**.

However, the killing of **phantom processes** can be disabled. The [`PhantomProcessList.trimPhantomProcessesIfNecessary()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/PhantomProcessList.java;l=416) uses the [`max_phantom_processes`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=172) setting for the [`activity_manager`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/DeviceConfig.java;l=67) namespace to decide how many processes to kill, which as mentioned defaults to `32`.

Based on the `trimPhantomProcessesIfNecessary()` `for` loop in source code, the `max_phantom_processes` value can be set to [`Integer.MAX_VALUE` (`2147483647`)](https://docs.oracle.com/javase/7/docs/api/java/lang/Integer.html#MAX_VALUE) to disable the killing of phantom processes, since the outer `if (mService.mConstants.MAX_PHANTOM_PROCESSES < mPhantomProcesses.size())` condition will always fail.

The `device_config` shell command can be used to change the default value as listed in the [Commands to disable phantom process killing and TLDR](#commands-to-disable-phantom-process-killing-and-tldr) section below. **It must be run with the `shell` (`adb`) or `root` user.** Check the [`device_config` command](#device_config-command) section below for more info on it.

The `device_config` command calls [`DeviceConfigService`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DeviceConfigService.java;l=270) to update the `max_phantom_processes` value, which triggers [`ActivityManagerConstants.updateMaxPhantomProcesses()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=1201) so that changes take effect.

After boot up, the [`ActivityManagerConstants.loadDeviceConfigConstants()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=820) called during `start()` will also read all properties set for the `activity_manager` device config namespace and call their relevant update functions in a `for` loop as per [`c85bbe00`](https://cs.android.com/android/_/android/platform/frameworks/base/+/c85bbe003d7fde1f2c60a192002981bc9bed9f21), including `updateMaxPhantomProcesses()`.

However, devices that have  after `3-4` mins have passed after device reboot and unlocked and after the `BOOT_COMPLETED` event is sent, all the settings for all namespaces, or at least `activity_manager` and `media` are reset and I only get following. Most namespaces don't have any properties set by default so I tested only `2`. Note that `BOOT_COMPLETED` event is not sent in `BFU` (before first unlock) state and lock screen needs to be unlocked for things to start and `BOOT_COMPLETED` will be sent after a few minutes after unlocking.

```
# Initial value after reboot after setting value
$ adb shell "/system/bin/device_config list activity_manager"
max_phantom_processes=2147483647
max_phantom_processes1=1
push_messaging_over_quota_behavior=0

# After few minutes of boot or whenever gms config update triggers 
$ adb shell "/system/bin/device_config list activity_manager"
push_messaging_over_quota_behavior=0
```

&nbsp;



## Commands to disable phantom process killing and TLDR

**Note:** Termux also supplies the [`android-tool`](https://github.com/termux/termux-packages/blob/master/packages/android-tools/build.sh) package that comes with the `adb` binary which you can use to run `adb` commands directly in termux app after you have set up `adb` wireless mode and connect to it. To install the package, run `pkg install android-tools`. Turning on `adb` wireless mode directly from device itself can only normally be done on Android `11` devices, to turn it on for lower Android versions requires connecting to a pc and running the `adb tcpip 5555` command on every boot. Check [Connect to a device over Wi-Fi (Android 11+)](https://developer.android.com/studio/command-line/adb#connect-to-a-device-over-wi-fi-android-11+) and [Connect to a device over Wi-Fi (Android 10 and lower)](https://developer.android.com/studio/command-line/adb#wireless) android docs for more info. Some more info in the [reddit announcement post](https://www.reddit.com/r/termux/comments/mmu2iu/announce_adb_is_now_packaged_for_termux/
).

### Android 12L, 13 and higher

**Run commands once** to disable killing of **phantom processes** and **processes using excessive cpu**.

  - `root`: `su -c "settings put global settings_enable_monitor_phantom_procs false"` or `su -c "setprop persist.sys.fflag.override.settings_enable_monitor_phantom_procs false"`
  - `adb`: `adb shell "settings put global settings_enable_monitor_phantom_procs false"`

Note that on debug android builds, like android virtual device Google API's releases, the setting can also be changed from `Android Settings` -> `System` -> `Developer Options` -> `Feature flags`. **On production devices, the `Feature flags` page will be empty.**

&nbsp;

Trying to run `setprop persist.sys.fflag.override.settings_enable_monitor_phantom_procs` with `adb` will [fail due to selinux restrictions](settings_enable_monitor_phantom_procs-with-adb.png) since a [sepolicy excemption](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:system/sepolicy/private/shell.te;l=149) does not exist for it like it does for other flags like [`settings_dynamic_system`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:system/sepolicy/private/property_contexts;l=71) used for [dynamic system updates](https://source.android.com/docs/core/ota/dynamic-system-updates#feature-flag). The [`FeatureFlagUtils.isEnabled()`](https://cs.android.com/android/platform/superproject/+/android-12.1.0_r5:frameworks/base/core/java/android/util/FeatureFlagUtils.java;l=94) gives priority to `settings` `global` value over `persist.sys.fflag`. Check https://twitter.com/MishaalRahman/status/1491487491026833413 and https://twitter.com/MishaalRahman/status/1491489385170227205

If you are currently using Termux app [github builds](https://github.com/termux/termux-app#github), you can open termux app and open left drawer -> `Settings icon` -> `About` and it should show the current value of `MONITOR_PHANTOM_PROCS` under `Software` info section on `Android 12+` devices and will be marked `<unsupported>` if its not supported in current android build. It should also show in next F-Droid release `v0.119.0`.

&nbsp;



### Android 12

On Android 12, **no setting exists** that can be changed with `adb` or `root` to disable killing of **processes using excessive cpu** and users will have to manually limit the cpu usage of forked processes using commands like [`nice`/`cpulimit`](https://unix.stackexchange.com/questions/151883/limiting-processes-to-not-exceed-more-than-10-of-cpu-usage). A [xposed module](https://github.com/rovo89/XposedBridge/wiki/Development-tutorial) for **rooted users** that can be run with [`Magisk`](https://github.com/topjohnwu/Magisk) and [`LSPosed`](https://github.com/LSPosed/LSPosed) that hooks into [`ActivityManagerService.checkExcessivePowerUsage()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=14237) to disable the killing is planned to be created.

To disable killing of **phantom processes**, read on.

If you have `com.google.android.gms` (google play services) installed on your device as system app, then it will overwrite all your config settings remotely on your device after `3-4` mins have passed after device boot up and screen unlocked and whenever gms config update comes, which may happen at any time of the day/week/month. To prevent that from happening you **must disable disable device config sync**, otherwise value will randomly reset and all excess phantom processes will immediately be killed.

You can check if `gms` is installed on your device and has the permission to overwrite setting, run below commands. If `android.permission.WRITE_DEVICE_CONFIG: granted=true` is returned, then your settings should be overridden. Other OEMs or device manufacturers may also have similar system services to update settings, and you will need to disable device config sync for those cases as well.

  - `root`: `su -c "/system/bin/dumpsys package com.google.android.gms | grep WRITE_DEVICE_CONFIG"`
  - `adb`: `adb shell "/system/bin/dumpsys package com.google.android.gms | grep WRITE_DEVICE_CONFIG"`

&nbsp;

#### If gms or related services do not exist on the device

Just set `max_phantom_processes` to `2147483647` to permanently disable killing of **phantom processes**.

  - `root`: `su -c "/system/bin/device_config put activity_manager max_phantom_processes 2147483647"`
  - `adb`: `adb shell "/system/bin/device_config put activity_manager max_phantom_processes 2147483647"`
  - Tasker `Adb Wifi` or `Run Shell` action with `Use Root` toogle enabled: `/system/bin/device_config put activity_manager max_phantom_processes 2147483647"`


#### If gms or related services do exist on the device

Disable device config sync permanently and set `max_phantom_processes` to `2147483647` to permanently disable killing of **phantom processes**.

**Use this at your own risk** and if you are willing to accept any known and unknown risks, like no recovery from boot loops, crashes and bad config values. Check `*device config sync*` and [Why device config may get reset?](#why-device-config-may-get-reset) sections below for more info. You can also run `/system/bin/device_config set_sync_disabled_for_tests until_reboot` to disable sync until next reboot, in which case you will need to run the commands on every reboot since `gms` will reset the config at boot.

  - `root`: `su -c "/system/bin/device_config set_sync_disabled_for_tests persistent; /system/bin/device_config put activity_manager max_phantom_processes 2147483647"`
  - `adb`: `adb shell "/system/bin/device_config set_sync_disabled_for_tests persistent; /system/bin/device_config put activity_manager max_phantom_processes 2147483647"`

&nbsp;

If you are currently using Termux app [github builds](https://github.com/termux/termux-app#github), you can open termux app and open left drawer -> `Settings icon` -> `About` and it should show the current value of `MAX_PHANTOM_PROCESSES` and `DEVICE_CONFIG_SYNC_DISABLED` under `Software` info section if termux has been granted `DUMP` and `PACKAGE_USAGE_STATS` permissions with `pm grant com.termux android.permission.DUMP; appops set --user 0 com.termux GET_USAGE_STATS allow` with `adb`/`root`. It should also show in next F-Droid release `v0.119.0`.

## &nbsp;



##### Re-enable device config sync

Disabling device config sync may prevent android from recovering from crashes, boot loops and bad configs since it will prevent `RescueParty` and `RollbackManager` to restore default settings, check [Why device config may get reset?](#why-device-config-may-get-reset) section below for details. You should ideally **enable syncing again before upgrading your android OS version**. If you make sure to always enable it before updating and current config is stable, then disabling the config "should" likely be safe. I don't know if sync value is preserved during an OS update. If it is preserved and if you forgot to enable syncing before the update, and update gets in a boot loop due to some issue, then you may be stuck in it if it was due to bad config, since android won't be able to restore default config.

  - `root`: `su -c "/system/bin/device_config set_sync_disabled_for_tests none"`
  - `adb`: `adb shell "/system/bin/device_config set_sync_disabled_for_tests none"`

&nbsp;



##### Check if device config sync is currently disabled

Setting single values will still be allowed if state is disabled.

###### Android 13+

The following commands should return `persistent` if sync permanently disabled or `until_reboot` if disabled till next boot. Default value is `none`.

  - `root`: `su -c "/system/bin/device_config get_sync_disabled_for_tests"`
  - `adb`: `adb shell "/system/bin/device_config get_sync_disabled_for_tests"`


###### Android 12

The following commands should return `true` if sync is disabled permanently or till next boot. Default value is `false`.

  - `root`: `su -c "/system/bin/device_config is_sync_disabled_for_tests"`
  - `adb`: `adb shell "/system/bin/device_config is_sync_disabled_for_tests"`

&nbsp;



##### Check device config sync mode stored in settings

The following commands should return `0` for `none`, `1` for `persistent` and also `0` for `until_reboot` since [actual value is stored in memory](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=1186) for it.

  - `root`: `su -c "/system/bin/settings get global device_config_sync_disabled"`
  - `adb`: `adb shell "/system/bin/settings get global device_config_sync_disabled"`


---

&nbsp;






## Why device config may get reset?

So I wasn't sure what was causing the reset of device configs or if it was just by android design. The other settings namespaces, `system`, `secure` and `global` weren't reset, only `config` was.

One of the ways that settings can be reset is by calling [`DeviceConfig.resetToDefaults()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/DeviceConfig.java;l=845) (`device_config reset`). Normally, that should be called by[`RescueParty`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/RescueParty.java;l=358), which help rescue the system from crash loops, but I didn't see any `Attempting rescue level` `logcat` entries, only `Starting to observe` ones and boot up succeeded anyways.

The other way is [`DeviceConfig.setProperties()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/DeviceConfig.java;l=796) which overrides the properties for the namespaces and removes any properties whose keys are not part of the properties objects passed. It calls [`Settings.setStrings()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/Settings.java;l=16289), which calls [`sNameValueCache.setStringsForPrefix()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/Settings.java;drc=android-12.0.0_r4;l=2829), which calls the [`CALL_METHOD_SET_ALL_CONFIG`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/Settings.java;l=2406) method set in [`mCallSetAllCommand`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/Settings.java;l=16213), which is then [received by `SettingsProvider`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=465) and which finally calls [`SettingsProvider.setAllConfigSettings()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=1142) to set all the properties. Now in this function in android `12` via [`ad3d45a`](https://cs.android.com/android/_/android/platform/frameworks/base/+/ad3d45a642e30bc5b653a0ee46a95348aa870a4f), the `isSyncDisabledConfigLocked()` option was added which if `true` would prevent all(bulk) settings from being set and the function would just return. This was done since if something tried to overwrite all the values while [cts tests](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/tests/coretests/src/android/provider/DeviceConfigTest.java;l=775) were running, they may fail due to mismatched settings set by the tests themselves.

The sync can be permanently disabled by running the `adb shell "/system/bin/device_config set_sync_disabled_for_tests persistent"` command once, which calls [`DeviceConfig.setSyncDisabled()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/DeviceConfig.java;l=851), which ultimately calls [`SettingsProvider.setSyncDisabledConfigLocked()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=1186) to store the mode in settings `global` namespace with the key `device_config_sync_disabled`.

The other modes are listed [here](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/Settings.java;l=16182) and in `device_config` command help (`device_config -h`).

The currently enforced value can be checked with `adb shell "/system/bin/device_config is_sync_disabled_for_tests"`. The current mode int can be checked with `adb shell "/system/bin/settings get global device_config_sync_disabled"`. It should return `0` for `none`, `1` for `persistent` and also `0` for `until_reboot` since [actual value is stored in memory](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=1186) for it.

I didn't know what was calling `DeviceConfig.setProperties()` at `BOOT_COMPLETED` event since this was likely what was being called since sync disabled check only exists only in this function (unless caller was manually checking sync mode first). It can be called by [`RescueParty.resetDeviceConfigForPackages()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/RescueParty.java;l=234) which is called by [`RollbackManager.commit()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/rollback/Rollback.java;l=610), but I didn't see any `logcat` entries or `"commitRollback id=`, logged by [`RollbackManagerServiceImpl.commitRollbackInternal()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/rollback/RollbackManagerServiceImpl.java;l=445). Only `RollbackManager: mRollbackLifetimeDurationInMillis=1209600000` is shown, which is logged by [`RollbackManagerServiceImpl.onBootCompleted()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/rollback/RollbackManagerServiceImpl.java;l=591).

Then I thought since android isn't doing it, something else must be and only `android` core, `com.google.android.gsf` (google services framework), `com.google.android.gms` (google play services) and `com.android.shell` packages have the `WRITE_DEVICE_CONFIG` permission (check [`The `device_config` command`](#device_config-command) section), so it must be that "damn" google doing it. So I decompiled the APKs and found that it had calls to `DeviceConfig.setProperties()` and the `logcat` also had entries for it after `BOOT_COMPLETED` event.

Note the `updateFromConfigurations` and `com.google.android.gms.platformconfigurator.UPDATE_IMMEDIATE` entries in the log which confirms that `gms` was responsible for updating the config by ultimately calling `atzo.c(str, hashMap)`, which calls `DeviceConfig.setProperties()`, which calls [`SettingsProvider.setAllConfigSettings()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=1155), which calls [`setConfigSettingsLocked()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=2995), which calls [`reportDeviceConfigUpdate()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=2337), which triggers the [`EXTRA_NAMESPACE_UPDATED_CALLBACK`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/Settings.java;l=2461) callback, which is then received by [`RescueParty`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/RescueParty.java;l=279) and it calls [`startObservingPackages()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/RescueParty.java;l=300). The `startObservingPackages()` logs the `RescueParty: Starting to observe: [android], updated namespace: activity_manager` entry and calls [`PackageWatchdog.startObservingHealth()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/PackageWatchdog.java;l=293), which starts monitoring the package's health so that if health checks fails, mitigations are done for recovery.

I searched on the internet about it and it seems that [`opengapps`](https://gitlab.opengapps.org/opengapps/all/-/issues/3) and [`vendor_gapps`](https://gitlab.com/razorloves/vendor_gapps/-/commit/267ae92773193830cf37ef9178a04bf967df4ac2) have denied `com.google.android.gms` the `WRITE_DEVICE_CONFIG` permission so that google doesn't remotely set device configurations on user devices (from the background). So users on such ROMs shouldn't have resetting problems caused by `gms`, unless they have their own ways.

I also don't know if `gms` downloads any new `config` updates from server and applies them even without a reboot. If it does, then disabling sync would be the way to go, since otherwise config may get updated at random times and processes getting killed.

Moreover, denying the `WRITE_DEVICE_CONFIG` permission to `gms` on non-rooted devices can't be done since its not a `runtime` or `development` permission and if you try to run `adb shell "pm revoke com.google.android.gms android.permission.WRITE_DEVICE_CONFIG"`, then [`PermissionManagerService`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java;l=1664) will throw `Exception occurred while executing 'revoke': java.lang.SecurityException: Permission android.permission.WRITE_DEVICE_CONFIG requested by com.google.android.gms is not a changeable permission type`.

I am not sure of the exact harm in disabling the sync in production devices permanently (`persistent` mode). At the very least it will cause issues with recovery from crashes and bad configs since it will prevent `RescueParty` and `RollbackManager` from doing its job. But I guess if the user has confirmed the current config is stable and working fine, it should probably be fine to set it to `persistent`. It should probably still be enabled again (`none` mode) before any OS updates are done.

However, if you don't want to take such risks, then keep sync enabled, and set the value few minutes after boot and run all your boot tasks/scripts after that.

<details>
<summary>gms log</summary>

```
11-04 01:17:14.471  1080  3235 I atzq    : updateFromConfigurations using legacy put method [CONTEXT service_id=204 ]
11-04 01:17:14.484  1385  1552 I Icing   : Indexing com.google.android.gms-apps from com.google.android.gms
11-04 01:17:14.487  1385  1552 I Icing   : Indexing done com.google.android.gms-apps
11-04 01:17:14.598  1385  3297 W Checkin : [CheckinRequestProcessor] Default Task : Checkin failed: https://android.googleapis.com/checkin (fragment #0) Rejected response from server: Bad Request
11-04 01:17:14.598  1385  3297 W Checkin : java.io.IOException: Rejected response from server: Bad Request
11-04 01:17:14.598  1385  3297 W Checkin :  at rgw.c(:com.google.android.gms@212423056@21.24.23 (190800-396046673):2)
11-04 01:17:14.598  1385  3297 W Checkin :  at rgw.a(:com.google.android.gms@212423056@21.24.23 (190800-396046673):54)
11-04 01:17:14.598  1385  3297 W Checkin :  at com.google.android.gms.checkin.CheckinIntentOperation.onHandleIntent(:com.google.android.gms@212423056@21.24.23 (190800-396046673):54)
11-04 01:17:14.598  1385  3297 W Checkin :  at com.google.android.chimera.IntentOperation.onHandleIntent(:com.google.android.gms@212423056@21.24.23 (190800-396046673):2)
11-04 01:17:14.598  1385  3297 W Checkin :  at roy.onHandleIntent(:com.google.android.gms@212423056@21.24.23 (190800-396046673):4)
11-04 01:17:14.598  1385  3297 W Checkin :  at eck.run(:com.google.android.gms@212423056@21.24.23 (190800-396046673):5)
11-04 01:17:14.598  1385  3297 W Checkin :  at ecj.run(:com.google.android.gms@212423056@21.24.23 (190800-396046673):11)
11-04 01:17:14.598  1385  3297 W Checkin :  at bvgk.run(:com.google.android.gms@212423056@21.24.23 (190800-396046673):2)
11-04 01:17:14.598  1385  3297 W Checkin :  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
11-04 01:17:14.598  1385  3297 W Checkin :  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
11-04 01:17:14.598  1385  3297 W Checkin :  at java.lang.Thread.run(Thread.java:920)
11-04 01:17:14.750  1080  3523 W .gms.persisten: Long monitor contention with owner [com.google.android.gms.chimera.PersistentIntentOperationService$ChimeraService-Executor] processing roy for action com.google.android.gms.platformconfigurator.UPDATE_IMMEDIATE (3235) at void eck.run()(:com.google.android.gms@212423056@21.24.23 (190800-396046673):5) waiters=0 in void eck.run() for 6.740s
11-04 01:17:14.756  3441  3441 E GooglePartnerSetup: Phenotype client.register: true
11-04 01:17:14.774   555  2266 I RescueParty: Starting to observe: [android], updated namespace: activity_manager
11-04 01:17:14.774   555   582 I PackageWatchdog: Syncing state, reason: observing new packages
11-04 01:17:14.774   555   582 D PackageWatchdog: rescue-party-observer added the following packages to monitor [android]
11-04 01:17:14.774   555   582 I PackageWatchdog: Syncing state, reason: updated observers
11-04 01:17:14.774   555   582 I PackageWatchdog: Not pruning observers, elapsed time: 0ms
11-04 01:17:14.774   555   582 I PackageWatchdog: Saving observer state to file
11-04 01:17:14.795  1826  1826 I HibernationPolicy: scheduleHibernationJob with frequency 1296000000ms and threshold 7776000000ms
11-04 01:17:14.797  1385  3297 I Checkin : [CheckinOperation] Checkin Operation finished with result: FAILURE finish time: 186901.
11-04 01:17:14.800   555   555 I PackageWatchdog: Syncing health check requests for packages: {}
11-04 01:17:14.800   555   555 I ExplicitHealthCheckController: Service not ready to get health check supported packages. Binding...
11-04 01:17:14.804   555   555 I ExplicitHealthCheckController: Explicit health check service is bound
11-04 01:17:14.807   555   555 I ExplicitHealthCheckController: Explicit health check service is connected ComponentInfo{com.google.android.ext.services/android.ext.services.watchdog.ExplicitHealthCheckServiceImpl}
11-04 01:17:14.807   555   555 I ExplicitHealthCheckController: Service initialized, syncing requests
11-04 01:17:14.807   555   555 I PackageWatchdog: Syncing health check requests for packages: {}
11-04 01:17:14.807   555   555 D ExplicitHealthCheckController: Getting health check supported packages
11-04 01:17:14.810   555  2070 I ExplicitHealthCheckController: Explicit health check supported packages [PackageConfig{com.google.android.networkstack, 86400000}]
11-04 01:17:14.810   555  2070 D PackageWatchdog: Received supported packages [PackageConfig{com.google.android.networkstack, 86400000}]
11-04 01:17:14.810   555  2070 D ExplicitHealthCheckController: Getting health check requested packages
11-04 01:17:14.811   555  2070 I ExplicitHealthCheckController: Explicit health check requested packages []
11-04 01:17:14.811   555  2070 I ExplicitHealthCheckController: No more health check requests, unbinding...
11-04 01:17:14.813   555  2070 I ExplicitHealthCheckController: Explicit health check service is unbound
```

</details>


---

&nbsp;


Note that `atzq.m(String str)` calls `atzo.c(str, hashMap)` which calls `DeviceConfig.setProperties(builder.build());`

<details>
<summary>gms classes</summary>

```java
package com.google.android.gms.platformconfigurator;

/* compiled from: :com.google.android.gms@212423056@21.24.23 (190800-396046673) */
/* loaded from: classes4.dex */
public class PhenotypeConfigurationUpdateListener extends IntentOperation {
   
    private final void f() {
        d();
        if (!ctkn.g() || d) {
            e();
        } else {
            Intent startIntent = IntentOperation.getStartIntent(this, PhenotypeConfigurationUpdateListener.class, "com.google.android.gms.platformconfigurator.UPDATE_IMMEDIATE");
            int i2 = ahxv.a;
            new txn(this).d("com.google.android.gms.platformconfigurator.UPDATE_IMMEDIATE", 3, ctkn.a.a().a(), ahxv.c(this, 0, startIntent, 67108864), null);
        }
        atzq b2 = atzm.b(this);
        synchronized (atzq.a) {
            b2.a();
            int i3 = atzo.a;
            boolean z = true;
            for (String str : ctkn.c().a) {
                z = b2.c(str) && z;
            }
            b2.c(null);
        }
        c = true;
    }

    /* JADX WARN: Can't fix incorrect switch cases order, some code will duplicate */
    @Override // com.google.android.chimera.IntentOperation
    public final void onHandleIntent(Intent intent) {
        char c2;
        if (atzp.b()) {
            String action = intent.getAction();
            switch (action.hashCode()) {
                case -1174223015:
                    if (action.equals("com.google.android.gms.platformconfigurator.UPDATE_IMMEDIATE")) {
...
                case -544318258:
                    if (action.equals("com.google.android.gms.phenotype.COMMITTED")) {
...
                case 438946629:
                    if (action.equals("com.google.android.gms.chimera.container.CONTAINER_UPDATED")) {
...
                case 798292259:
                    if (action.equals("android.intent.action.BOOT_COMPLETED")) {
...
                case 832939294:
                    if (action.equals("com.google.android.gms.platformconfigurator.RETRY_UPDATE")) {
...
                case 2019499159:
                    if (action.equals("com.google.android.gms.phenotype.UPDATE")) {
...
            }
    }
}
```

```java
package defpackage;

/* compiled from: :com.google.android.gms@212423056@21.24.23 (190800-396046673) */
/* renamed from: atzq  reason: default package */
/* loaded from: classes4.dex */
public final class atzq {
   
    private final String m(String str) {
        Configurations i = i(atzp.a(str), null);
        if (i == null) {
            return null;
        }
        HashMap hashMap = new HashMap();
        for (Map.Entry entry : i.e.entrySet()) {
            Flag[] flagArr = ((Configuration) entry.getValue()).b;
            for (Flag flag : flagArr) {
                hashMap.put(flag.b, flag.c());
            }
        }
        try {
            if (!atzo.c(str, hashMap)) {
                return null;
            }
            return i.a;
        } catch (SecurityException e) {
            i.c(b.j(), "setProperties failed with SecurityException", 5069, e);
            return null;
        }
    }

...

    final boolean g(int i, Configurations configurations, String str, String str2) {
        try {
            if (str2 != null) {
                bxqb bxqb = (bxqb) b.h();
                bxqb.W(5077);
                bxqb.t("updateFromConfigurations DeviceConfig for namespace %s", str2);
                p(i, configurations, str, str2);
                return true;
            }
            bxqb bxqb2 = (bxqb) b.h();
            bxqb2.W(5075);
            bxqb2.p("updateFromConfigurations using legacy put method");
            p(i, configurations, str, null);
            return true;
        } catch (SecurityException e) {
            i.c(b.j(), "updateFromConfigurations failed with SecurityException", 5076, e);
            return false;
        }
    }
}
```

```java
package defpackage;

import android.provider.DeviceConfig;
import java.util.Map;

/* compiled from: :com.google.android.gms@212423056@21.24.23 (190800-396046673) */
/* renamed from: atzo  reason: default package */
/* loaded from: classes4.dex */
public final class atzo {
    public static final /* synthetic */ int a = 0;
    private static final uep b = uep.e(tty.PLATFORM_CONFIGURATOR);
    private static boolean c = false;

    public static String a(String str, String str2) {
        return DeviceConfig.getProperty(str, str2);
    }

    public static void b(String str, String str2, String str3, boolean z) {
        DeviceConfig.setProperty(str, str2, str3, z);
    }

    /* JADX INFO: Access modifiers changed from: package-private */
    public static boolean c(String str, Map map) {
        try {
            DeviceConfig.Properties.Builder builder = new DeviceConfig.Properties.Builder(str);
            for (Map.Entry entry : map.entrySet()) {
                builder.setString((String) entry.getKey(), (String) entry.getValue());
            }
            return DeviceConfig.setProperties(builder.build());
        } catch (Exception e) {
            if (e instanceof DeviceConfig.BadConfigException) {
                throw new atzn(e);
            } else if (e instanceof SecurityException) {
                throw ((SecurityException) e);
            } else {
                throw new RuntimeException(e);
            }
        } catch (NoClassDefFoundError e2) {
            i.c(b.j(), "This device does not support atomic writes. Falling back to writing individual flags", 5055, e2);
            c = true;
            return false;
        }
    }
}
```

</details>


---

&nbsp;





## What are the risks with disabling phantom process killing?

Users may want to know if there are any risks in setting the value to `2147483647`. I don't see any in my limited view, specially considering that no max limit was defined by android devs during the commit, but they may not have considered someone setting a value this high. But then again, there was no limit in any previous android version, so only additional useless book keeping of phantom processing would just be occurring and battery drainage consistent with older android versions. If there is something I am missing, it would be great if someone pointed it out.

You can also monitor battery usage of apps or their CPU usage with `root` or `adb` with `top -m 10` command, or monitor the processes created with `ps -efww` command. You can also check [`google's battery-historian`](https://github.com/google/battery-historian) for getting battery usage stats of devices.

**Note that completely disabling phantom killing will allow apps going crazy with background processes to consume lot of battery and android should provide per app exemption instead, so that only specific apps can be exempted and phantom killing still applies to other apps.**


---

&nbsp;





## How to manually trigger phantom process killing in Termux?

Install the termux apps from Github or F-Droid as per [installation instructions](https://github.com/termux/termux-app#installation). F-Droid direct link is https://f-droid.org/en/packages/com.termux. After installing the app, open it and let bootstrap installation to finish.

Run the following command and put termux in background and connect or disconnect the charger.

```
for i in $(seq 40); do
sha256sum /dev/zero &
done
```

You can monitor cpu usage with `top` command and run `killall sha256sum` to kill the processes started.


When you return to Termux app, it should say `[Process completed (signal 9) - press Enter]` because the main `login` `bash` shell should have gotten killed and any other older processes above `32`. Sometimes  `bash` gets killed as soon as one runs the loop while Termux is still in foreground, sometimes after switching to another app and come back, and sometimes takes a few minutes depending on when phantom process killing is scheduled. The connecting or disconnecting the charger works instantly, as per `Battery power changes` trigger mentioned in the `How phantom process killing gets scheduled?` section.

Note that `bash` was started with `pid` `5683` and `logcat` logs the killing. The `sha256sum` commands below are CPU intensive but only their count matters for the killing. The `bash` process would barely use any CPU but still get killed first since its the oldest. Check the `How are phantom processes killed?` section for more info on how killing is prioritized.

```
I ActivityManager: Killing PhantomProcessRecord {9ccd0ab 5683:5641:bash/u0a147}: Trimming phantom processes
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5683 in 0ms
I ActivityManager: Process PhantomProcessRecord {9ccd0ab 5683:5641:bash/u0a147} died
```


<details>
<summary>Transcript to start 1 bash process + 40 sha256sum processes</summary>

```
~ $ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
u0_a147   5641   340  5  1970 ?        00:00:02 com.termux
u0_a147   5683  5641  0  1970 pts/0    00:00:00 /data/data/com.termux/files/usr/bin/bash
u0_a147   5728  5683  1  1970 pts/0    00:00:00 ps -ef
~ $ for i in $(seq 40); do sha256sum /dev/zero & done
[1] 5731
[2] 5732
[3] 5733
[4] 5734
[5] 5735
[6] 5736
[7] 5737
[8] 5738
[9] 5739
[10] 5740
[11] 5741
[12] 5742
[13] 5743
[14] 5744
[15] 5745
[16] 5746
[17] 5747
[18] 5748
[19] 5749
[20] 5750
[21] 5751
[22] 5752
[23] 5753
[24] 5754
[25] 5755
[26] 5756
[27] 5757
[28] 5758
[29] 5759
[30] 5760
[31] 5761
[32] 5762
[33] 5763
[34] 5764
[35] 5765
[36] 5766
[37] 5767
[38] 5768
[39] 5769
[40] 5770
~ $
[Process completed (signal 9) - press Enter]
```
</details>

<details>
<summary>Logcat showing 9 phantom processes trimmed</summary>

```
I ActivityManager: Killing PhantomProcessRecord {9ccd0ab 5683:5641:bash/u0a147}: Trimming phantom processes
I ActivityManager: Killing PhantomProcessRecord {30b2308 5731:5641:sha256sum/u0a147}: Trimming phantom processes
I ActivityManager: Killing PhantomProcessRecord {9a8aa1 5732:5641:sha256sum/u0a147}: Trimming phantom processes
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5683 in 0ms
I ActivityManager: Killing PhantomProcessRecord {19044c6 5733:5641:sha256sum/u0a147}: Trimming phantom processes
I ActivityManager: Killing PhantomProcessRecord {685cc87 5734:5641:sha256sum/u0a147}: Trimming phantom processes
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5731 in 0ms
I ActivityManager: Killing PhantomProcessRecord {2b367b4 5735:5641:sha256sum/u0a147}: Trimming phantom processes
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5732 in 0ms
I ActivityManager: Killing PhantomProcessRecord {9342fdd 5736:5641:sha256sum/u0a147}: Trimming phantom processes
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5733 in 0ms
I ActivityManager: Killing PhantomProcessRecord {3bbe752 5737:5641:sha256sum/u0a147}: Trimming phantom processes
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5734 in 0ms
I ActivityManager: Killing PhantomProcessRecord {2e8aa23 5738:5641:sha256sum/u0a147}: Trimming phantom processes
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5735 in 0ms
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5736 in 0ms
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5737 in 0ms
I libprocessgroup: Successfully killed process cgroup uid 10147 pid 5738 in 0ms
I ActivityManager: Process PhantomProcessRecord {9ccd0ab 5683:5641:bash/u0a147} died
I ActivityManager: Process PhantomProcessRecord {2e8aa23 5738:5641:sha256sum/u0a147} died
I init    : Untracked pid 5738 received signal 9
I ActivityManager: Process PhantomProcessRecord {9342fdd 5736:5641:sha256sum/u0a147} died
I ActivityManager: Process PhantomProcessRecord {685cc87 5734:5641:sha256sum/u0a147} died
I ActivityManager: Process PhantomProcessRecord {19044c6 5733:5641:sha256sum/u0a147} died
I init    : Untracked pid 5734 received signal 9
I init    : Untracked pid 5736 received signal 9
I init    : Untracked pid 5733 received signal 9
I ActivityManager: Process PhantomProcessRecord {30b2308 5731:5641:sha256sum/u0a147} died
I init    : Untracked pid 5731 received signal 9
I ActivityManager: Process PhantomProcessRecord {9a8aa1 5732:5641:sha256sum/u0a147} died
I ActivityManager: Process PhantomProcessRecord {2b367b4 5735:5641:sha256sum/u0a147} died
I ActivityManager: Process PhantomProcessRecord {3bbe752 5737:5641:sha256sum/u0a147} died
I init    : Untracked pid 5732 received signal 9
I init    : Untracked pid 5735 received signal 9
I init    : Untracked pid 5737 received signal 9
```
</details>

<details>
<summary>Remaining 32 tracked phantom processes</summary>

```
  All Active App Child Processes:
    proc #0: PhantomProcessRecord {c1b29f4 5739:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5739 ppid=5641 knownSince=-56s278ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #1: PhantomProcessRecord {511651d 5740:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5740 ppid=5641 knownSince=-56s278ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #2: PhantomProcessRecord {6a51392 5741:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5741 ppid=5641 knownSince=-56s278ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #3: PhantomProcessRecord {52c0163 5742:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5742 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #4: PhantomProcessRecord {1baf160 5743:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5743 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #5: PhantomProcessRecord {2643619 5744:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5744 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #6: PhantomProcessRecord {d7cd6de 5745:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5745 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #7: PhantomProcessRecord {55350bf 5746:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5746 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #8: PhantomProcessRecord {9cdc38c 5747:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5747 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #9: PhantomProcessRecord {8791ad5 5748:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5748 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #10: PhantomProcessRecord {77b82ea 5749:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5749 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #11: PhantomProcessRecord {c7639db 5750:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5750 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #12: PhantomProcessRecord {e598c78 5751:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5751 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #13: PhantomProcessRecord {1bd8f51 5752:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5752 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #14: PhantomProcessRecord {906e3b6 5753:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5753 ppid=5641 knownSince=-56s277ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #15: PhantomProcessRecord {61498b7 5754:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5754 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #16: PhantomProcessRecord {1d6f824 5755:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5755 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #17: PhantomProcessRecord {7facf8d 5756:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5756 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #18: PhantomProcessRecord {5158542 5757:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5757 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #19: PhantomProcessRecord {bd00953 5758:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5758 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #20: PhantomProcessRecord {39d7290 5759:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5759 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #21: PhantomProcessRecord {e51d789 5760:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5760 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #22: PhantomProcessRecord {57ab38e 5761:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5761 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #23: PhantomProcessRecord {5c7e7af 5762:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5762 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #24: PhantomProcessRecord {f0f27bc 5763:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5763 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #25: PhantomProcessRecord {ef76345 5764:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5764 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #26: PhantomProcessRecord {bf27a9a 5765:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5765 ppid=5641 knownSince=-56s276ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #27: PhantomProcessRecord {4b54fcb 5766:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5766 ppid=5641 knownSince=-56s275ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #28: PhantomProcessRecord {18503a8 5767:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5767 ppid=5641 knownSince=-56s275ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #29: PhantomProcessRecord {8afeec1 5768:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5768 ppid=5641 knownSince=-56s275ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #30: PhantomProcessRecord {7eda666 5769:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5769 ppid=5641 knownSince=-56s275ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
    proc #31: PhantomProcessRecord {1c71da7 5770:5641:sha256sum/u0a147}
      user #0 uid=10147 pid=5770 ppid=5641 knownSince=-56s275ms killed=false
      lastCpuTime=0 oom adj=0 seq=13
```

</details>

<details>
<summary>App and  Device Info</summary>

### Termux App Info

**APP_NAME**: `Termux`  
**PACKAGE_NAME**: `com.termux`  
**VERSION_NAME**: `0.117`  
**VERSION_CODE**: `117`  
**UID**: `10147`  
**TARGET_SDK**: `28`  
**IS_DEBUGGABLE_BUILD**: `true`  
**APK_RELEASE**: `Github`  
**SIGNING_CERTIFICATE_SHA256_DIGEST**: `B6DA01480EEFD5FBF2CD3771B8D1021EC791304BDD6C4BF41D3FAABAD48EE5E1`  
##

### Device Info

#### Software

**OS_VERSION**: `5.10.43-android12-9-00031-g02d62d5cece1-ab7792588`  
**SDK_INT**: `31`  
**RELEASE**: `12`  
**ID**: `SE1A.211012.001`  
**DISPLAY**: `sdk_gphone64_x86_64-userdebug 12 SE1A.211012.001 7818354 dev-keys`  
**INCREMENTAL**: `7818354`  
**SECURITY_PATCH**: `2021-11-05`  
**IS_DEBUGGABLE**: `1`  
**IS_EMULATOR**: `1`  
**IS_TREBLE_ENABLED**: `true`  
**TYPE**: `userdebug`  
**TAGS**: `dev-keys`  

#### Hardware

**MANUFACTURER**: `Google`  
**BRAND**: `google`  
**MODEL**: `sdk_gphone64_x86_64`  
**PRODUCT**: `sdk_gphone64_x86_64`  
**BOARD**: `goldfish_x86_64`  
**HARDWARE**: `ranchu`  
**DEVICE**: `emulator64_x86_64_arm64`  
**SUPPORTED_ABIS**: `x86_64, arm64-v8a`  

</details>


---

&nbsp;





## McAfee app gone crazy

Now, as per [OP's `dumpsys activity processes` log](https://github.com/termux/termux-app/issues/2366#issuecomment-955582861), the `com.wsandroid.suite`'s `logcat` processes were taking **ALL** `32` spots for max phantom processes and they all had [`oom adj=100`/`VISIBLE_APP_ADJ = 100`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=248), which may or may not have been the same as the app process itself as used by `mTempPhantomProcesses` sorter. The termux process likely had [`PERCEPTIBLE_APP_ADJ = 200`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=244) being a foreground service. Considering that it was `termux` `bash` that got killed, we know that the `com.wsandroid.suite` **app process** had lower oom ajd than termux app process and that is why recently started `bash` got killed and not the `logcat` ones running for almost `4hrs` (`knownSince=-3h57m28s448ms`). If termux had same oom adj as `com.wsandroid.suite`, then `bash` should have survived since it was younger than most `logcat` processes. The `bash` as per my current tests has `oom_score_adj=0` while termux is in background, buts its oom value is not considered.

<details>
<summary>dumpsys activity processes log</summary>
<p>

```
ACTIVITY MANAGER RUNNING PROCESSES (dumpsys activity processes)
  OOM levels:
    -900: SYSTEM_ADJ (   73,728K)
    -800: PERSISTENT_PROC_ADJ (   73,728K)
    -700: PERSISTENT_SERVICE_ADJ (   73,728K)
      0: FOREGROUND_APP_ADJ (   73,728K)
     100: VISIBLE_APP_ADJ (   92,160K)
     200: PERCEPTIBLE_APP_ADJ (  110,592K)
     225: PERCEPTIBLE_MEDIUM_APP_ADJ (  129,024K)
     250: PERCEPTIBLE_LOW_APP_ADJ (  129,024K)
     300: BACKUP_APP_ADJ (  221,184K)
     400: HEAVY_WEIGHT_APP_ADJ (  221,184K)
     500: SERVICE_ADJ (  221,184K)
     600: HOME_APP_ADJ (  221,184K)
     700: PREVIOUS_APP_ADJ (  221,184K)
     800: SERVICE_B_ADJ (  221,184K)
     900: CACHED_APP_MIN_ADJ (  221,184K)
     999: CACHED_APP_MAX_ADJ (  322,560K)

  Process OOM control (74 total, non-act at 11, non-svc at 11):

  mHomeProcess: ProcessRecord{22364da 21734:com.google.android.apps.nexuslauncher/u0a61}
  mPreviousProcess: ProcessRecord{1dac4a6 15558:com.termux/u0a383}


  Process LRU list (sorted by oom_adj, 74 total, non-act at 11, non-svc at 11):

  All Active App Child Processes:
    proc #0: PhantomProcessRecord {8f0d7dc 2611:3461:logcat/u0a164}
      user #0 uid=10164 pid=2611 ppid=3461 knownSince=-1h35m23s39ms killed=false
      lastCpuTime=10 timeUsed=+20ms oom adj=100 seq=665
    proc #1: PhantomProcessRecord {7920e5 3162:3461:logcat/u0a164}
      user #0 uid=10164 pid=3162 ppid=3461 knownSince=-1h27m21s601ms killed=false
      lastCpuTime=10 timeUsed=+10ms oom adj=100 seq=665
    proc #2: PhantomProcessRecord {1b6f7ba 3253:3461:logcat/u0a164}
      user #0 uid=10164 pid=3253 ppid=3461 knownSince=-1h27m21s601ms killed=false
      lastCpuTime=10 timeUsed=+30ms oom adj=100 seq=665
    proc #3: PhantomProcessRecord {e3e566b 4120:3461:logcat/u0a164}
      user #0 uid=10164 pid=4120 ppid=3461 knownSince=-1h26m23s909ms killed=false
      lastCpuTime=10 timeUsed=+30ms oom adj=100 seq=665
    proc #4: PhantomProcessRecord {a195c8 5790:3461:logcat/u0a164}
      user #0 uid=10164 pid=5790 ppid=3461 knownSince=-1h15m15s965ms killed=false
      lastCpuTime=10 timeUsed=0 oom adj=100 seq=665
    proc #5: PhantomProcessRecord {11fe661 5893:3461:logcat/u0a164}
      user #0 uid=10164 pid=5893 ppid=3461 knownSince=-1h14m56s494ms killed=false
      lastCpuTime=10 timeUsed=+10ms oom adj=100 seq=665
    proc #6: PhantomProcessRecord {2ccd586 8968:3461:logcat/u0a164}
      user #0 uid=10164 pid=8968 ppid=3461 knownSince=-41m37s575ms killed=false
      lastCpuTime=10 timeUsed=+10ms oom adj=100 seq=665
    proc #7: PhantomProcessRecord {5e9ee47 9531:3461:logcat/u0a164}
      user #0 uid=10164 pid=9531 ppid=3461 knownSince=-38m38s396ms killed=false
      lastCpuTime=10 timeUsed=+10ms oom adj=100 seq=665
    proc #8: PhantomProcessRecord {d1e4674 10169:3461:logcat/u0a164}
      user #0 uid=10164 pid=10169 ppid=3461 knownSince=-28m49s512ms killed=false
      lastCpuTime=10 timeUsed=0 oom adj=100 seq=665
    proc #9: PhantomProcessRecord {113879d 10534:3461:logcat/u0a164}
      user #0 uid=10164 pid=10534 ppid=3461 knownSince=-28m49s513ms killed=false
      lastCpuTime=10 timeUsed=0 oom adj=100 seq=665
    proc #10: PhantomProcessRecord {910c412 10626:3461:logcat/u0a164}
      user #0 uid=10164 pid=10626 ppid=3461 knownSince=-28m49s513ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #11: PhantomProcessRecord {fcd27e3 11349:3461:logcat/u0a164}
      user #0 uid=10164 pid=11349 ppid=3461 knownSince=-21m29s836ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #12: PhantomProcessRecord {91355e0 11406:3461:logcat/u0a164}
      user #0 uid=10164 pid=11406 ppid=3461 knownSince=-21m29s836ms killed=false
      lastCpuTime=10 timeUsed=0 oom adj=100 seq=665
    proc #13: PhantomProcessRecord {ca60099 12372:3461:logcat/u0a164}
      user #0 uid=10164 pid=12372 ppid=3461 knownSince=-21m29s836ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #14: PhantomProcessRecord {6330f5e 12666:3461:logcat/u0a164}
      user #0 uid=10164 pid=12666 ppid=3461 knownSince=-14m56s274ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #15: PhantomProcessRecord {3205f3f 12782:3461:logcat/u0a164}
      user #0 uid=10164 pid=12782 ppid=3461 knownSince=-14m56s273ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #16: PhantomProcessRecord {b87f00c 13866:3461:logcat/u0a164}
      user #0 uid=10164 pid=13866 ppid=3461 knownSince=-10m49s764ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #17: PhantomProcessRecord {94d0d55 14132:3461:logcat/u0a164}
      user #0 uid=10164 pid=14132 ppid=3461 knownSince=-3h58m28s795ms killed=false
      lastCpuTime=10 timeUsed=+30ms oom adj=100 seq=665
    proc #18: PhantomProcessRecord {49cc36a 14317:3461:logcat/u0a164}
      user #0 uid=10164 pid=14317 ppid=3461 knownSince=-10m49s764ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #19: PhantomProcessRecord {845b05b 14461:3461:logcat/u0a164}
      user #0 uid=10164 pid=14461 ppid=3461 knownSince=-9m45s82ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #20: PhantomProcessRecord {2ca00f8 14838:3461:logcat/u0a164}
      user #0 uid=10164 pid=14838 ppid=3461 knownSince=-5m49s707ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #21: PhantomProcessRecord {41e29d1 14910:3461:logcat/u0a164}
      user #0 uid=10164 pid=14910 ppid=3461 knownSince=-3h57m28s448ms killed=false
      lastCpuTime=10 timeUsed=+30ms oom adj=100 seq=665
    proc #22: PhantomProcessRecord {a3bac36 14938:3461:logcat/u0a164}
      user #0 uid=10164 pid=14938 ppid=3461 knownSince=-3h57m28s448ms killed=false
      lastCpuTime=10 timeUsed=+30ms oom adj=100 seq=665
    proc #23: PhantomProcessRecord {1a4f737 15447:3461:logcat/u0a164}
      user #0 uid=10164 pid=15447 ppid=3461 knownSince=-2m53s747ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #24: PhantomProcessRecord {21a34a4 15725:3461:logcat/u0a164}
      user #0 uid=10164 pid=15725 ppid=3461 knownSince=-1m25s518ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #25: PhantomProcessRecord {c0a920d 15874:3461:logcat/u0a164}
      user #0 uid=10164 pid=15874 ppid=3461 knownSince=-49s617ms killed=false
      lastCpuTime=0 oom adj=100 seq=665
    proc #26: PhantomProcessRecord {c0e55c2 17826:3461:logcat/u0a164}
      user #0 uid=10164 pid=17826 ppid=3461 knownSince=-3h29m58s826ms killed=false
      lastCpuTime=10 timeUsed=+30ms oom adj=100 seq=665
    proc #27: PhantomProcessRecord {647cfd3 27443:3461:logcat/u0a164}
      user #0 uid=10164 pid=27443 ppid=3461 knownSince=-2h11m45s994ms killed=false
      lastCpuTime=10 timeUsed=+30ms oom adj=100 seq=665
    proc #28: PhantomProcessRecord {e17f710 29010:3461:logcat/u0a164}
      user #0 uid=10164 pid=29010 ppid=3461 knownSince=-2h4m57s282ms killed=false
      lastCpuTime=10 timeUsed=+40ms oom adj=100 seq=665
    proc #29: PhantomProcessRecord {3db4209 30904:3461:logcat/u0a164}
      user #0 uid=10164 pid=30904 ppid=3461 knownSince=-1h53m27s378ms killed=false
      lastCpuTime=10 timeUsed=+20ms oom adj=100 seq=665
    proc #30: PhantomProcessRecord {e700c0e 31295:3461:logcat/u0a164}
      user #0 uid=10164 pid=31295 ppid=3461 knownSince=-1h51m7s727ms killed=false
      lastCpuTime=10 timeUsed=+30ms oom adj=100 seq=665
    proc #31: PhantomProcessRecord {a35962f 32065:3461:logcat/u0a164}
      user #0 uid=10164 pid=32065 ppid=3461 knownSince=-1h50m23s158ms killed=false
      lastCpuTime=20 timeUsed=+20ms oom adj=100 seq=665

  mUidChangeDispatchCount=39302
  Slow UID dispatches:
    com.android.server.job.controllers.QuotaController$QcUidObserver: 0 / Max 4ms
    android.app.IUidObserver$Stub$Proxy: 0 / Max 8ms
    android.app.IUidObserver$Stub$Proxy: 0 / Max 6ms
    android.app.ActivityManager$UidObserver: 0 / Max 5ms
    android.app.ActivityManager$UidObserver: 0 / Max 11ms
    android.app.IUidObserver$Stub$Proxy: 1 / Max 161ms
    android.app.IUidObserver$Stub$Proxy: 0 / Max 10ms
    com.android.server.AppStateTrackerImpl$UidObserver: 0 / Max 3ms
    android.app.IUidObserver$Stub$Proxy: 3 / Max 37ms
    com.android.server.pm.ShortcutService$4: 0 / Max 5ms
    com.android.server.power.hint.HintManagerService$UidObserver: 0 / Max 14ms
    com.android.server.usage.UsageStatsService$3: 0 / Max 9ms
    android.app.ActivityManager$UidObserver: 4 / Max 210ms
    com.android.server.net.NetworkPolicyManagerService$4: 0 / Max 10ms
    com.android.server.job.JobSchedulerService$4: 0 / Max 17ms
    com.android.server.job.controllers.QuotaController$QcUidObserver: 1 / Max 25ms
    com.android.server.PinnerService$3: 0 / Max 4ms
    com.android.server.vibrator.VibrationSettings$UidObserver: 0 / Max 5ms
  mDeviceIdleAllowlist=[1000, 1001, 2000, 10008, 10016, 10022, 10026, 10033, 10041, 10043, 10074, 10164, 10175, 10187, 10199, 10206, 10214, 10281, 10283, 10328, 10383]
  mDeviceIdleExceptIdleAllowlist=[1000, 1001, 2000, 10008, 10009, 10016, 10018, 10022, 10026, 10033, 10034, 10039, 10041, 10043, 10045, 10052, 10053, 10067, 10074, 10088, 10136, 10152, 10164, 10175, 10187, 10199, 10206, 10214, 10281, 10283, 10328, 10352, 10383]
  mDeviceIdleTempAllowlist=[10016, 10057]
  mFgsStartTempAllowList:
    u0a57:  duration=10000 callingUid=u0a57 reasonCode=ALARM_MANAGER_WHILE_IDLE reason=broadcast:u0a57:settings.intelligence.battery.action.PERIODIC_JOB_UPDATE,reason: expiration=2021-10-30 19:00:10.026 (+9s602ms)
  mForceBackgroundCheck=false
```
</details>


---

&nbsp;



I asked OP to start Termux and McAfee apps and then switching to some other app (to avoid `PERCEPTIBLE_RECENT_FOREGROUND_APP_ADJ`) and then waiting for a while and then running `adb shell "/system/bin/dumpsys activity processes | grep -E -A 50 'ProcessRecord\{.*com.termux'"` and `adb shell "/system/bin/dumpsys activity processes | grep -E -A 50 'ProcessRecord\{.*com.wsandroid.suite'"` to get `ProcessRecord`s of both apps. The commands were run at a random time and not after trimming but it shows that Termux app process had oom adj `cur=200` and McAfee app process had `cur=100`. So these are likely the normal oom adj while in background, which would result in **Termux processes being killed first during any trimming** in which both apps are in background, which is what has been happening on OP's device frequently. The lower oom of McAfee app process (which is also same as `logcat` processes oom) is likely due to additional connections and providers.

<details>
<summary>Termux ProcessRecord</summary>

```
$ adb shell "/system/bin/dumpsys activity processes | grep -E -A 50 'ProcessRecord\{.*com.termux'"
  *APP* UID 10383 ProcessRecord{f26c83c 18144:com.termux/u0a383}
    user #0 uid=10383 gids={3003, 50383, 20383, 9997}
    mRequiredAbi=arm64-v8a instructionSet=arm64
    class=com.termux.app.TermuxApplication
    dir=/data/app/~~gQtDuMCK77ed6DPdCmeWAA==/com.termux-MfgXPeqANrvwr29oJKqfNA==/base.apk publicDir=/data/app/~~gQtDuMCK77ed6DPdCmeWAA==/com.termux-MfgXPeqANrvwr29oJKqfNA==/base.apk data=/data/user/0/com.termux
    packageList={com.termux}
    compat={440dpi}
    thread=android.app.IApplicationThread$Stub$Proxy@f4fbbd
    pid=18144
    lastActivityTime=-53s621ms
    startSeq=4028
    mountMode=DEFAULT
    lastPssTime=-59s212ms pssProcState=4 pssStatType=1 nextPssTime=+4m4s71ms
    lastPss=20MB lastSwapPss=379KB lastCachedPss=0.00 lastCachedSwapPss=0.00 lastRss=57MB
    trimMemoryLevel=0
    procStateMemTracker: best=1 (1=1 7.59375x)
    lastRequestedGc=-9m30s398ms lastLowMemory=-9m30s398ms reportLowMemory=false
    reportedInteraction=false fgInteractionTime=-50s903ms
    adjSeq=647062 lruSeq=194591
    oom adj: max=1001 curRaw=200 setRaw=200 cur=200 set=200
    mCurSchedGroup=2 setSchedGroup=2 systemNoUi=false
    curProcState=4 mRepProcState=4 setProcState=4 lastStateTime=-9m30s394ms
    curCapability=-CMN setCapability=-CMN
    allowStartFgsState=4
    hasShownUi=true pendingUiClean=true
    cached=false empty=true
    lastTopTime=-50s903ms
    lastInvisibleTime=2021-11-01 20:53:37.304 (-50s903ms)
    hasStartedServices=true
    mHasForegroundServices=true forcingToImportant=null
    Services:
      - ServiceRecord{ecab86c u0 com.termux/.app.TermuxService}
    mConnections:
      - ConnectionRecord{44ab2ca u0 com.termux/.app.TermuxService:@99e35}
    Published Providers:
      - com.termux.filepicker.TermuxDocumentsProvider
        -> ContentProviderRecord{e30f5b2 u0 com.termux/.filepicker.TermuxDocumentsProvider}
      - com.termux.app.TermuxOpenReceiver$ContentProvider
        -> ContentProviderRecord{f610903 u0 com.termux/.app.TermuxOpenReceiver$ContentProvider}
    Connected Providers:
      - 8f1e68c/com.android.providers.settings/.SettingsProvider->18144:com.termux/u0a383 s1/1 u0/0 +9m30s337ms
    lastCompactTime=0 lastCompactAction=0
    isFreezeExempt=false isPendingFreeze=false isFrozen=false
    Activities:
      - ActivityRecord{aa40b10 u0 com.termux/.app.TermuxActivity t270}
    Recent Tasks:
      - Task{eb62c28 #270 type=standard A=10383:com.termux U=0 visible=false mode=fullscreen translucent=true sz=1}
    BoundClientUids:[10383]
     Configuration={1.0 310mcc150mnc [en_US,es_US,nb_NO] ldltr sw392dp w392dp h767dp 440dpi nrml long port night finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2220) mAppBounds=Rect(0, 0 - 1080, 2176) mMaxBounds=Rect(0, 0 - 1080, 2220) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} as.2 s.6012 fontWeightAdjustment=0}
     OverrideConfiguration={1.0 310mcc150mnc [en_US,es_US,nb_NO] ldltr sw392dp w392dp h767dp 440dpi nrml long port night finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2220) mAppBounds=Rect(0, 0 - 1080, 2176) mMaxBounds=Rect(0, 0 - 1080, 2220) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} as.2 s.1 fontWeightAdjustment=0}
     mLastReportedConfiguration={1.0 310mcc150mnc [en_US,es_US,nb_NO] ldltr sw392dp w392dp h767dp 440dpi nrml long port night finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2220) mAppBounds=Rect(0, 0 - 1080, 2176) mMaxBounds=Rect(0, 0 - 1080, 2220) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} as.2 s.6012 fontWeightAdjustment=0}
```

</details>


<details>
<summary>McAfee ProcessRecord</summary>

```
$ adb shell "/system/bin/dumpsys activity processes | grep -E -A 50 'ProcessRecord\{.*com.wsandroid.suite'"
  *APP* UID 10164 ProcessRecord{24e34c1 3461:com.wsandroid.suite/u0a164}
    user #0 uid=10164 gids={3002, 3003, 3001, 50164, 20164, 9997}
    mRequiredAbi=arm64-v8a instructionSet=arm64
    class=com.wsandroid.suite.MMSApplication
    manageSpaceActivityName=com.wavesecure.activities.ManageSpaceActivity
    dir=/data/app/~~iWO3mRIXmB_uMRsr0gJvYw==/com.wsandroid.suite-IQCqZvqHzO_TB7Dqs4PPsw==/base.apk publicDir=/data/app/~~iWO3mRIXmB_uMRsr0gJvYw==/com.wsandroid.suite-IQCqZvqHzO_TB7Dqs4PPsw==/base.apk data=/data/user/0/com.wsandroid.suite
    packageList={com.wsandroid.suite}
    packageDependencies={com.google.android.webview}
    compat={440dpi}
    thread=android.app.IApplicationThread$Stub$Proxy@2d1b295
    pid=3461
    lastActivityTime=-1m3s99ms
    startSeq=41
    mountMode=DEFAULT
    lastPssTime=-1m14s752ms pssProcState=4 pssStatType=1 nextPssTime=+58m44s597ms
    lastPss=210MB lastSwapPss=41MB lastCachedPss=0.00 lastCachedSwapPss=0.00 lastRss=220MB
    trimMemoryLevel=0
    procStateMemTracker: best=1 (1=1 Infinityx)
    lastRequestedGc=-15h6m1s363ms lastLowMemory=-15h10m20s35ms reportLowMemory=false
    reportedInteraction=false fgInteractionTime=-58s379ms
    adjSeq=647074 lruSeq=194598
    oom adj: max=1001 curRaw=100 setRaw=100 cur=100 set=100
    mCurSchedGroup=2 setSchedGroup=2 systemNoUi=false
    curProcState=4 mRepProcState=4 setProcState=4 lastStateTime=-2d8h39m55s697ms
    curCapability=LCMN setCapability=LCMN
    allowStartFgsState=4
    hasShownUi=true pendingUiClean=true
    cached=false empty=true
    notCachedSinceIdle=true initialIdlePss=81684
    lastTopTime=-58s378ms
    lastInvisibleTime=2021-11-01 20:53:45.546 (-58s385ms)
    hasStartedServices=true
    mHasForegroundServices=true forcingToImportant=null
    Services:
      - ServiceRecord{15396e2 u0 com.wsandroid.suite/com.mcafee.monitor.MMSAccessibilityService}
      - ServiceRecord{45bf48f u0 com.wsandroid.suite/com.mcafee.messaging.google.GoogleFCMMessageReceiver}
      - ServiceRecord{a1e649e u0 com.wsandroid.suite/com.mcafee.sdk.wp.core.siteadvisor.service.SiteAdvisorService}
      - ServiceRecord{c0ba211 u0 com.wsandroid.suite/com.mcafee.broadcast.MMSSystemBroadcastListencerService}
    mConnections:
      - ConnectionRecord{1036c33 u0 CR com.wsandroid.suite/org.chromium.content.app.SandboxedProcessService0:0:@1ff27a2}
      - ConnectionRecord{109c957 u0 CR WACT CAPS com.google.android.gms/.chimera.PersistentDirectBootAwareApiService:@305f2d6}
      - ConnectionRecord{6cc1bee u0 CR WPRI com.wsandroid.suite/org.chromium.content.app.SandboxedProcessService0:0:@ba02569}
      - ConnectionRecord{b9567ee u0 CR IMP com.wsandroid.suite/com.mcafee.messaging.google.GoogleFCMMessageReceiver:@3cf2169}
    Published Providers:
      - androidx.lifecycle.ProcessLifecycleOwnerInitializer
        -> ContentProviderRecord{bf09faa u0 com.wsandroid.suite/androidx.lifecycle.ProcessLifecycleOwnerInitializer}
      - com.mcafee.core.provider.KeyValueDataProvider
        -> ContentProviderRecord{d95f79b u0 com.wsandroid.suite/com.mcafee.core.provider.KeyValueDataProvider}
      - com.mcafee.core.provider.JSONDataContentProvider
        -> ContentProviderRecord{72dd738 u0 com.wsandroid.suite/com.mcafee.core.provider.JSONDataContentProvider}
      - com.moengage.core.internal.storage.database.MoEProvider
```

</details>


---

&nbsp;



I also asked OP to check what `logcat` processes McAfee was actually running by running the command `adb shell "ps -efww | grep logcat"` and it reported that it was running fricking `135` `logcat` processes for the command: `logcat -v time -b main -b system -b events PackageManager:V ActivityManager:V SearchDialog:V AppSecurityPermissions:V MiniModeService:D *:S`. So many are running because OP set `max_phantom_processes` to `2147483647` and so they weren't being *culled* to default `32`. I emailed McAfee on `Oct, 31` at ` mmsfeedback@mcafee.com` but haven't received a "human" response. They definitely should fix this. [mcafee-logcat.txt](mcafee-logcat.txt)


---

&nbsp;





## The "unfair" advantage for foreground oriented apps

Note that popular apps that are by design foreground apps, like social media apps, browsers, etc will often have lower oom adj, [`FOREGROUND_APP_ADJ = 0`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=258) or [`PERCEPTIBLE_RECENT_FOREGROUND_APP_ADJ = 50`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=254) if they are being used or were recently used, and will **often** have "unfair" advantage over apps that do most of their work in background, like Tasker and for some users Termux as well, which with their foreground service notification will usually be at oom adj [`PERCEPTIBLE_APP_ADJ = 200`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=244). **An app that mostly works in background does not mean its unimportant**. Moreover, an app like Termux doesn't have any need to bind to additional services and create connections and shouldn't suffer because of it. Apps may try to do that unnecessarily now just to increase their oom adj values.


---

&nbsp;





## Buggy and malicious apps


> I am pretty sure the app normally wouldn't need to run that many processes and is likely "leaking processes",

Thinking more on this, any malicious or buggy app could just spawn `> 32` processes that don't consume much CPU and hinder other apps from running even a single process for long if the app process has a lower oom adj value at the time of phantom process trimming, like for example, running the `logcat` process, which doesn't consume much CPU.

Imagine, if some popular app like Google or Facebook app or any popular app from play store or a system app has leaking processes and is usually at a lower oom than other apps or is in foreground or last used while the *culling* of phantom processes gets scheduled, it would result in processes of other apps that may be far more important getting killed and being prevented from even be run for long, resulting in data loss. Others apps may just be trying to run a single process instead of `32`, but still get killed. Such apps would already have the "unfair advantage" and then waste even more spots in the `32` limit because of leaks. OEMs may set their own apps at lower oom adj too that also run phantom processes. The `com.google.android.gms` is normally at `100` oom adj too. How are app devs supposed to deal with that?

Now, this can be abused by a malicious app as well deliberately, possibly by starting a foreground service and running lot of useless processes. *Culling* will kill processes of apps that may not have a foreground service and were just running a short command in a receiver or something. The malicious app may not gain much from this, unless somehow affects a system process, but still harms other normal apps.


---

&nbsp;





## Improvements in heuristics and exemptions for apps

Neither trimming nor excessive CPU killing, nor empty process killing (including possibly daemon, like `sshd`, [termux-app/#2015](https://github.com/termux/termux-app/issues/2015), [termux-boot/#5](https://github.com/termux/termux-boot/issues/5#issuecomment-353601099)) should be done for apps that have **battery optimizations disabled**. A user explicitly granted an app the permission to "use battery", android shouldn't be killing its processes, unless under memory pressure, or at the very least shouldn't prioritize it. That should be like the whole point of disabling battery optimizations. Adding this to android OS should solve most of the concerns.

And if trimming needs to be done, the heuristics should improve. It should have some **minimum allowed limit of phantom processes for each app below which they are all ignored** and also a **max allowed limit per app** that should be something like `max_phantom_processes/3` (or whatever the android overlords think is ideal) above which trimming of the phantom processes of that specific app is prioritized before starting to kill processes of other apps, so that foreground oriented, malicious and buggy apps don't affect other apps too much.

If I hadn't asked OP for the logs, we wouldn't have known that McAfee was causing so much trouble for other apps and who knows how long it has been doing it. And OP was actually capable of reporting and providing logs, but an average user wouldn't be, and from their perspective, their apps would just be getting killed because had knowingly or blindly installed a buggy app from playstore. And even if they reported (without logs), devs wouldn't be able to reproduce the issue on their device, since they wouldn't likely have a buggy app installed, so this kinda issues will be harder to solve for devs and will have nothing to do with their code. The OS should protect users from that, even without them having to check `logcat` and `dumpsys` output.

Moreover, **total `32` processes limit of all apps combined is too low**. That can easily be hit during what one would consider normal conditions and not excessive. Just running `2-3` terminal sessions, each with optional terminal multiplexer (`tmux`) and then actually running some commands in them or just a script that divides work with background jobs (`command &`), plus some processes started incrementally as or for plugins, like `termux-api` or `termux-tasker` and you are easily looking at more than `15` to `30` processes for just Termux. Now add some user running some servers in Termux or add Tasker with its profiles and tasks, plus any other app, and `32` limit is hit instantly and processes getting trimmed, and some important work or data being lost.

---

&nbsp;










# Cached and Empty Processes

Cached and empty processes are killed by [`OomAdjuster.updateAndTrimProcessLSP()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java;l=1195) depending on whether their process state is [`PROCESS_STATE_CACHED_ACTIVITY`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/app/ProcessStateEnum.aidl;l=83) or [`PROCESS_STATE_CACHED_EMPTY`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/app/ProcessStateEnum.aidl;l=94). The difference between [cached](https://developer.android.com/guide/components/activities/process-lifecycle) and [empty](https://stuff.mit.edu/afs/sipb/project/android/docs/guide/components/processes-and-threads.html) app processes is that cached processes are tied to a component of the app like an `Activity` and empty processes are not. Forked child (phantom) processes of app processes are not considered, but are killed along with the app process. An app process with a foreground service will not be considered as an empty process either.

## Cache Settings

The amount of cached and empty processes that are killed or kept is based on `5` device config values.

- [`CUR_MAX_CACHED_PROCESSES`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=538) - The maximum number of cached processes that are kept. Defaults to [`mCustomizedMaxCachedProcesses = 32`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/res/res/values/config.xml;l=5046) which is read from android framework resources used for defining values for different hardware and product builds.
- [`CUR_MAX_EMPTY_PROCESSES`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=542) - The maximum number of empty processes that are kept. Defaults to `CUR_MAX_CACHED_PROCESSES / 2 = 16`.
- [`MAX_CACHED_PROCESSES`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=251) - Used for setting `CUR_TRIM_*` values. Defaults to [`DEFAULT_MAX_CACHED_PROCESSES = 32`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=115).
- [`CUR_TRIM_EMPTY_PROCESSES`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=546) - The number of empty processes that are allowed to be kept and are not necessary to be trimmed. Defaults to `(MAX_CACHED_PROCESSES / 4) = 8`.
- [`CUR_TRIM_CACHED_PROCESSES`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=550) - The number of cached processes that are allowed to be kept and are not necessary to be trimmed. Defaults to `(MAX_CACHED_PROCESSES - (MAX_CACHED_PROCESSES / 2)) / 3 = 5`.

The `CUR_MAX_CACHED_PROCESSES` and `CUR_MAX_EMPTY_PROCESSES` values are initialized by `ActivityManagerConstants` in the [constructor](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=793) and can be updated by sending a property update event for the `DeviceConfig` [`KEY_MAX_CACHED_PROCESSES = "max_cached_processes"`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=170) key, like with `device_config` command, which when [received](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=663) calls [`updateMaxCachedProcesses()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=1146). The values set in the constructor will be overridden with the `max_cached_processes` value in device config if its set when `start()` calls [`loadDeviceConfigConstants()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=820).

The `CUR_MAX_CACHED_PROCESSES` value can be overridden by the first valid value in the following list as defined by [`updateMaxCachedProcesses()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=1150).

- The [`mOverrideMaxCachedProcesses`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=526) value if its `>= 0`. It defaults to `-1`. It can be changed with the [`Background process limit`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:packages/apps/Settings/src/com/android/settings/development/BackgroundProcessLimitPreferenceController.java;l=90) developer option in android settings, which calls [`setOverrideMaxCachedProcesses()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=825).
- The `DeviceConfig` `max_cached_processes` key value if its set. It is not set by default.
- The `mCustomizedMaxCachedProcesses` value which is loaded from android resources. It defaults to `32`.

The `MAX_CACHED_PROCESSES` value is hard coded and cannot be updated and since `CUR_TRIM_EMPTY_PROCESSES` and `CUR_TRIM_CACHED_PROCESSES` are based on it, neither can they, as mentioned in [this comment](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=1162).


---

&nbsp;





## Why max limit for KILLING?

Now, you may be wondering why this this (low) `CUR_MAX_CACHED_PROCESSES` limit is used by android. It's cause of **MUSIC!!!** Yes, cause of Music! It was added in the [`8633e68`](https://cs.android.com/android/_/android/platform/frameworks/base/+/8633e68ebdf215f721834f7aa16c2f3cef1bae86
) commit with the following message.

> Music sometimes stops playing when navigation talks
>
>When a service transitions from foreground to background, we now push it
>to the top of the LRU list.  Also fix the activity manager to take care
>of killing processes if we go beyond a reasonable number of background
>process to keep around.

The commit defined the first value for `MAX_HIDDEN_APPS`/`CUR_MAX_CACHED_PROCESSES` and partially has the same comment that still [exists today](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=529). Now, this comment was added on `2010-04-23`, which is like android `~2.2` era, so this "large amounts of RAM" is kinda funny. What was considered large at the time? The [Google Nexus One](https://www.gsmarena.com/htc_google_nexus_one-3069.php) had `512MB` `RAM`! So was like `1GB` large??? Now `swap` files on class `6` external sd cards, those were the days! And now we have [Asus ROG Phone 5 Ultimate](https://www.gsmarena.com/asus_rog_phone_5_ultimate-10785.php) with `18 GB` `RAM`, I wonder if they followed the AOSP "guidelines" or if they just went batshit crazy! ;)

```
// The maximum number of hidden processes we will keep around before
// killing them; this is just a control to not let us go too crazy with
// keeping around processes on devices with large amounts of RAM.
static final int MAX_HIDDEN_APPS = 15;
```


---

&nbsp;





## How are cached and empty processes killed?

The values for `CUR_*_PROCESSES` that are set are used by `OomAdjuster` in [`assignCachedAdjIfNecessary()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java;l=1010) and [`updateAndTrimProcessLSP()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java;l=1142) to decide how many least recently used (LRU) cached and empty processes to kill/keep. You can read the [android docs for `OomAdjuster`](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/am/OomAdjuster.md) for more info.

The `updateAndTrimProcessLSP()`, kills any empty processes `> CUR_MAX_EMPTY_PROCESSES(16)` regardless of how long they have been running. The killed processes `logcat` entry will have `empty #<X>` where `X` is `CUR_MAX_EMPTY_PROCESSES+1` and is incremented for each additional processes that is killed after it.

Any processes `> CUR_TRIM_EMPTY_PROCESSES(8)` and `<= CUR_MAX_EMPTY_PROCESSES(16)` will get killed if they have been running for [`> ProcessList.MAX_EMPTY_TIME(30 mins)`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=295). The `MAX_EMPTY_TIME` value is hard coded and cannot be modified. This is done by the [`numEmpty > mConstants.CUR_TRIM_EMPTY_PROCESSES && app.getLastActivityTime() < oldTime`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java;l=1196) check, where [`oldTime = now - ProcessList.MAX_EMPTY_TIME`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java;l=862). The `logcat` entries for such process will have `empty for <X>s`, where `X` will be seconds from `now` since its been running, like `1800s`, which is equal to `30mins`.

The [`app.mLastActivityTime`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessRecord.java;l=282) is set by [`ProcessList.updateLruProcessLSP()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=3496) which is called to adjust its `LRU` priority when

- App process is [created](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=6219).
- App process is [attached](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=4535).
- `Service` [is started or its state changed](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;l=3919).
- `Broadcast` [delivered](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java;l=308).
- `ContentProvider` [connection created](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ContentProviderHelper.java;l=1360).
- [`ProcessRecord` info is updated](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessRecord.java;l=1240), which may be done when [activity is started or destroyed](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java;l=4890), by window process controller when [starting activity](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/wm/WindowProcessController.java;l=1168) or [updating process info](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/wm/WindowProcessController.java;l=1107), or by [activity task controller](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/wm/Task.java;l=6265).


---

&nbsp;





## How to check cache settings?

#### Check `MAX_CACHED_PROCESSES` value used by `ActivityManagerConstants`.

The `dumpsys activity settings` command [dumps the `MAX_CACHED_PROCESSES`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=1216) value with the name `max_cached_processes`. Note that this value **is not the** value for `DeviceConfig` [`KEY_MAX_CACHED_PROCESSES = "max_cached_processes"`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=170) key. The `MAX_CACHED_PROCESSES` value is not overridden in [`updateMaxCachedProcesses()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=1146).

  - `root`: `su -c "/system/bin/dumpsys activity settings | grep max_cached_processes"`
  - `adb`: `adb shell "/system/bin/dumpsys activity settings | grep max_cached_processes"`



#### Check `max_cached_processes` value stored in device config.

The value returned by the command will be `null` by default if not set with `device_config put` command.

  - `root`: `su -c "/system/bin/device_config get activity_manager max_cached_processes"`
  - `adb`: `adb shell "/system/bin/device_config get activity_manager max_cached_processes"`



#### Check `CUR_*_PROCESSES` value used by `ActivityManagerConstants`

  - `root`: `su -c "/system/bin/dumpsys activity settings | grep -B 2 -A 5 mCustomizedMaxCachedProcesses"`
  - `adb`: `adb shell "/system/bin/dumpsys activity settings | grep -B 2 -A 5 mCustomizedMaxCachedProcesses"`



---

&nbsp;





## What are default cache settings?

### ASOP

The default is `32` for `CUR_MAX_CACHED_PROCESSES` and `16` for `CUR_MAX_EMPTY_PROCESSES ` in android `12` AOSP and avd. The `CUR_MAX_CACHED_PROCESSES` value is set to `mCustomizedMaxCachedProcesses` since `max_cached_processes` value is `null` and `mOverrideMaxCachedProcesses` is `-1` ([not dumped if `< 0`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=1336)).

```
$ adb shell "/system/bin/dumpsys activity settings | grep max_cached_processes"
  max_cached_processes=32

$ adb shell "/system/bin/device_config get activity_manager max_cached_processes"
null

$ adb shell "/system/bin/dumpsys activity settings | grep -B 2 -A 5 mCustomizedMaxCachedProcesses"
...
mCustomizedMaxCachedProcesses=32
CUR_MAX_CACHED_PROCESSES=32
CUR_MAX_EMPTY_PROCESSES=16
CUR_TRIM_EMPTY_PROCESSES=8
CUR_TRIM_CACHED_PROCESSES=5
OOMADJ_UPDATE_QUICK=true
```

Android `12` avd `logcat` has the following entries. That shows that `empty` numbers from `17` onward, which is consistent with value of `CUR_MAX_EMPTY_PROCESSES=16` in the dump.

```
ActivityManager: Killing 879:com.android.settings/1000 (adj 945): empty #17
ActivityManager: Killing 2826:android.process.media/u0a58 (adj 945): empty #18
ActivityManager: Killing 1398:com.android.providers.calendar/u0a72 (adj 945): empty #19
ActivityManager: Killing 1153:com.android.dialer/u0a102 (adj 945): empty #20
ActivityManager: Killing 2937:com.android.imsserviceentitlement/u0a96 (adj 955): empty #21
ActivityManager: Killing 1227:android.process.acore/u0a56 (adj 955): empty #22
ActivityManager: Killing 3012:com.android.managedprovisioning/u0a61 (adj 955): empty #23
ActivityManager: Killing 2997:com.android.dynsystem:dynsystem/1000 (adj 955): empty #24
ActivityManager: Killing 2980:com.android.dynsystem/1000 (adj 965): empty #25
ActivityManager: Killing 2921:com.android.contacts/u0a101 (adj 965): empty #26
ActivityManager: Killing 2825:com.google.android.settings.intelligence/u0a99 (adj 965): empty #27
ActivityManager: Killing 2897:com.android.camera2/u0a113 (adj 965): empty #28
ActivityManager: Killing 2782:com.android.traceur/u0a63 (adj 975): empty #29
ActivityManager: Killing 2438:com.google.android.gms.unstable/u0a94 (adj 975): empty #30
ActivityManager: Killing 2590:com.google.android.youtube/u0a119 (adj 975): empty #31
ActivityManager: Killing 2548:com.google.android.tts/u0a126 (adj 975): empty #32
ActivityManager: Killing 1600:com.google.process.gapps/u0a94 (adj 985): empty #33
ActivityManager: Killing 2034:com.android.localtransport/1000 (adj 985): empty #34
```
&nbsp;



### Pixel Devices

The default is `64` for `CUR_MAX_CACHED_PROCESSES` and `32` for `CUR_MAX_EMPTY_PROCESSES ` in android `12` pixel 3a and Pixel 5. The `CUR_MAX_CACHED_PROCESSES` value is set to `max_cached_processes` since its set and `mOverrideMaxCachedProcesses` is `-1` ([not dumped if `< 0`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=1336)).

Note that the default value for `max_cached_processes` is not `null` because it was likely sent via a `gms` config update. The [`android-12.0.0_r1` `SP1A.210812.015` source code for `redfin`](https://android.googlesource.com/device/google/redfin) does not override [`config_customizedMaxCachedProcesses`](https://android.googlesource.com/device/google/redfin/+/refs/tags/android-12.0.0_r1/redfin/overlay/frameworks/base/core/res/res/values/config.xml) to `64`, which is also shown by the dump. However, `raven` (Pixel 6 Pro) does [override the value to `64`](https://android.googlesource.com/device/google/raviole/+/refs/tags/android-12.0.0_r4/raven/overlay/frameworks/base/core/res/res/values/config.xml#207).
```
$ adb shell "/system/bin/dumpsys activity settings | grep max_cached_processes"
  max_cached_processes=32

$ adb shell "/system/bin/device_config get activity_manager max_cached_processes"
max_cached_processes=64

$ adb shell "/system/bin/dumpsys activity settings | grep -B 2 -A 5 mCustomizedMaxCachedProcesses"
...
  mCustomizedMaxCachedProcesses=32
  CUR_MAX_CACHED_PROCESSES=64
  CUR_MAX_EMPTY_PROCESSES=32
  CUR_TRIM_EMPTY_PROCESSES=8
  CUR_TRIM_CACHED_PROCESSES=5
  OOMADJ_UPDATE_QUICK=true
```

The Pixel 5 belonging to @xeffyr has the following `logcat`  entries. That shows that `empty` numbers from `33` onward, which is consistent with value of `CUR_MAX_EMPTY_PROCESSES=32` in the dump.

```
ActivityManager: Killing 14312:com.google.android.apps.docs/u0a171 (adj 975): empty #33
ActivityManager: Killing 14286:com.google.android.documentsui/u0a66 (adj 975): empty #34
ActivityManager: Killing 14513:com.google.android.apps.wallpaper/u0a197 (adj 995): empty for 1800s
```


---

&nbsp;





## How to increase amount of cached and empty processes kept?

You can set a higher custom value for `max_cached_processes` device config, which will be set to `CUR_MAX_CACHED_PROCESSES` and whatever is passed, will be divided by `2` and set to `CUR_MAX_EMPTY_PROCESSES`.

Now, based on killing logic in [`updateAndTrimProcessLSP()`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java;l=1142), increasing `CUR_MAX_EMPTY_PROCESSES` value will only allow you too keep empty processes for max [`ProcessList.MAX_EMPTY_TIME(30 mins)`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=295). However, android `13` has provided `max_empty_time_millis` device config setting to allow changing `MAX_EMPTY_TIME` with [`57c0df8`](https://cs.android.com/android/_/android/platform/frameworks/base/+/57c0df88b7e50aded88dedbcbabf918138997256). The commit also adds supports to prevent killing of cached/empty processes before device unlock at boot for user `0` with `no_kill_cached_processes_until_boot_completed` (default: `true`) and within specific mins after unlock of any user with `no_kill_cached_processes_post_boot_completed_duration_millis` (default: `10mins`), the defaults were changed in [`1a4e911`](https://cs.android.com/android/_/android/platform/frameworks/base/+/1a4e911f53b4f554915a69289c7e2e0a5ad9f00c). To change `max_empty_time_millis`, run `device_config put activity_manager max_empty_time_millis <millis>`, where `<millis>` is the new timeout, like `3600000` for `60min`.

Moreover, OEMs may have their own killers too, like [`OppoClearSystemService`](https://github.com/termux/termux-app/issues/2015#issuecomment-860152470), so processes may still get killed on such devices that don't follow AOSP implementations.


To set a custom value for `max_cached_processes`, run something like following commands. Don't go too crazy and increase it too much [as advised by AOSP](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=529).

### Android `>= 10`

The `device_config` command was added in [android `10`](The-device-config-command) and for older versions these commands will not work.

- `root`: `su -c "/system/bin/device_config put activity_manager max_cached_processes 64"`
- `adb`: `adb shell "/system/bin/device_config put activity_manager max_cached_processes 64"`


You can also use `content` command to set the values.

```
adb shell "content call --uri content://settings/config --method LIST_config"
adb shell "content call --uri content://settings/config --method LIST_config | tr , '\n' | grep activity_manager/"
adb shell "content call --uri content://settings/config --method GET_config --arg 'activity_manager/max_cached_processes'"
adb shell "content call --uri content://settings/config --method PUT_config --arg 'activity_manager/max_cached_processes' --extra 'value:s:64'"
adb shell "content call --uri content://settings/config --method DELETE_config --arg 'activity_manager/max_cached_processes'"
```
&nbsp;



### Android `8.0 - 9`

Even though the `device_config` command did not exist in android `< 10` to store values in the `settings` `config` namespace, in android `8.0` support was added via [`0ef403e5`](https://cs.android.com/android/_/android/platform/frameworks/base/+/0ef403e53e2762d077750dd0a50b73c2125cadb0) for storing `max_cached_processes` and some other constants in the `settings` `global` namespace under a single `activity_manager_constants` key as a comma separated list of `key=value` pairs. This design is still used for some components, like `job_scheduler_quota_controller`, which you can check with `adb shell "settings get global job_scheduler_quota_controller_constants"`.

Now, since this is single key storing all the other keys, you can't just run `settings put` command to set a value, since it will overwrite all the existing values.

You can first get the default value with `adb shell "settings get global activity_manager_constants"`, then update or append `,max_cached_processes=64` to the default value and then put the joint value back with `adb shell "settings get global activity_manager_constants '<joint_value>'"`. You can probably use some regex to do it too.

When the property will be updated, the [`ActivityManagerConstants.updateConstants()`](https://cs.android.com/android/platform/superproject/+/android-8.0.0_r51:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=240) function will get called, which will use [`KeyValueListParser(',')`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/util/KeyValueListParser.java) [`mParser`](https://cs.android.com/android/platform/superproject/+/android-8.0.0_r51:frameworks/base/services/core/java/com/android/server/am/ActivityManagerConstants.java;l=183) to read each value and update it.

I haven't tested this since I only own an Android `7` device, which does not support this `max_cached_processes` and only has hardcoded values like [`ProcessList.MAX_CACHED_APPS`](https://cs.android.com/android/platform/superproject/+/android-7.0.0_r1:frameworks/base/services/core/java/com/android/server/am/ProcessList.java;l=147), nor do I need it since managing the `RAM` with `minfree` values and `swap` file works just fine for me and background child processes run for hours without getting killed.


---

&nbsp;





## `device_config` command

The [`device_config`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DeviceConfigService.java;l=270) command was added in android `10` via [`1278d1c7`](https://cs.android.com/android/_/android/platform/frameworks/base/+/1278d1c7363e211f706098f06c6c7c9d88a1c47d). It uses a separate `config` namespace in addition to the old `system`, `secure` and `global` namespaces controlled by `settings` command. They are all managed by android's [`Settings`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/Settings.java;l=2393) and [`SettingsProvider`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java).

To [set](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/provider/DeviceConfig.java;l=790
) `config` settings requires [`root` or `shell` (adb) user](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DeviceConfigService.java;l=345) or [`WRITE_DEVICE_CONFIG`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/res/AndroidManifest.xml;l=3143) permission, which has `protectionLevel` `signature|verifier|configurator` and so cannot be granted over `adb` to apps since protection level does not include `development`.

The various `protectionLevel` values are listed in the documentation [here](https://developer.android.com/reference/android/R.attr#protectionLevel) which is sourced from [`attrs_manifest.xml`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/res/res/values/attrs_manifest.xml;l=179). The flags are defined by [`PermissionInfo`](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r4:frameworks/base/core/java/android/content/pm/PermissionInfo.java).

The `WRITE_DEVICE_CONFIG` permission by default is granted to `android` core, `com.google.android.gsf` (google services framework), `com.google.android.gms` (google play services) and `com.android.shell` (package for `shell` user, used for commands run with `adb`).

**Note:** Termux also supplies the [`android-tool`](https://github.com/termux/termux-packages/blob/master/packages/android-tools/build.sh) package that comes with the `adb` binary which you can use to run `adb` commands directly in termux app after you have set up `adb` wireless mode and connect to it. To install the package, run `pkg install android-tools`. Turning on `adb` wireless mode directly from device itself can only normally be done on Android `11` devices, to turn it on for lower Android versions requires connecting to a pc and running the `adb tcpip 5555` command on every boot. Check [Connect to a device over Wi-Fi (Android 11+)](https://developer.android.com/studio/command-line/adb#connect-to-a-device-over-wi-fi-android-11+) and [Connect to a device over Wi-Fi (Android 10 and lower)](https://developer.android.com/studio/command-line/adb#wireless) android docs for more info. Some more info in the [reddit announcement post](https://www.reddit.com/r/termux/comments/mmu2iu/announce_adb_is_now_packaged_for_termux/
).

<details>
<summary>permission info</summary>

```
$ adb shell dumpsys package
...
  Package [android] (8c12d02):
    userId=1000
    sharedUser=SharedUserSetting{cdb4e0b android.uid.system/1000}
    declared permissions:
      android.permission.DUMP: prot=signature|privileged|development, INSTALLED
      android.permission.READ_LOGS: prot=signature|privileged|development, INSTALLED
      android.permission.PACKAGE_USAGE_STATS: prot=signature|privileged|development|appop|retailDemo, INSTALLED
      android.permission.READ_DEVICE_CONFIG: prot=signature|preinstalled, INSTALLED
      android.permission.WRITE_DEVICE_CONFIG: prot=signature|verifier|configurator, INSTALLED
      android.permission.WRITE_SECURE_SETTINGS: prot=signature|privileged|development, INSTALLED
      android.permission.WRITE_SETTINGS: prot=signature|appop|pre23|preinstalled, INSTALLED
...
  SharedUser [com.google.uid.shared] (603bbad):
    userId=10094
    Packages
      PackageSetting{5aa6e67 com.google.android.gsf/10094}
      PackageSetting{67cc226 com.google.android.gms/10094}
    install permissions:
      android.permission.WRITE_DEVICE_CONFIG: granted=true
...
  SharedUser [android.uid.shell] (6931355):
    userId=2000
    Packages
      PackageSetting{9870e0a com.android.shell/2000}
    install permissions:
      android.permission.WRITE_DEVICE_CONFIG: granted=true
```

</details>

&nbsp;

<details>
<summary>device_config command help Android 12</summary>

```
$ adb shell "/system/bin/device_config -h"
Device Config (device_config) commands:
  help
      Print this help text.
  get NAMESPACE KEY
      Retrieve the current value of KEY from the given NAMESPACE.
  put NAMESPACE KEY VALUE [default]
      Change the contents of KEY to VALUE for the given NAMESPACE.
      {default} to set as the default value.
  delete NAMESPACE KEY
      Delete the entry for KEY for the given NAMESPACE.
  list [NAMESPACE]
      Print all keys and values defined, optionally for the given NAMESPACE.
  reset RESET_MODE [NAMESPACE]
      Reset all flag values, optionally for a NAMESPACE, according to RESET_MODE.
      RESET_MODE is one of {untrusted_defaults, untrusted_clear, trusted_defaults}
      NAMESPACE limits which flags are reset if provided, otherwise all flags are reset
  set_sync_disabled_for_tests SYNC_DISABLED_MODE
      Modifies bulk property setting behavior for tests. When in one of the disabled modes this ensures that config isn't overwritten.
      SYNC_DISABLED_MODE is one of:
        none: Sync is not disabled. A reboot may be required to restart syncing.
        persistent: Sync is disabled, this state will survive a reboot.
        until_reboot: Sync is disabled until the next reboot.
  is_sync_disabled_for_tests
      Prints 'true' if sync is disabled, 'false' otherwise.
```

</details>

&nbsp;

<details>
<summary>device_config command help Android 13</summary>

```
adb shell "/system/bin/device_config -h"
Device Config (device_config) commands:
  help
      Print this help text.
  get NAMESPACE KEY
      Retrieve the current value of KEY from the given NAMESPACE.
  put NAMESPACE KEY VALUE [default]
      Change the contents of KEY to VALUE for the given NAMESPACE.
      {default} to set as the default value.
  delete NAMESPACE KEY
      Delete the entry for KEY for the given NAMESPACE.
  list [NAMESPACE]
      Print all keys and values defined, optionally for the given NAMESPACE.
  reset RESET_MODE [NAMESPACE]
      Reset all flag values, optionally for a NAMESPACE, according to RESET_MODE.
      RESET_MODE is one of {untrusted_defaults, untrusted_clear, trusted_defaults}
      NAMESPACE limits which flags are reset if provided, otherwise all flags are reset
  set_sync_disabled_for_tests SYNC_DISABLED_MODE
      Modifies bulk property setting behavior for tests. When in one of the disabled modes this ensures that config isn't overwritten.
      SYNC_DISABLED_MODE is one of:
        none: Sync is not disabled. A reboot may be required to restart syncing.
        persistent: Sync is disabled, this state will survive a reboot.
        until_reboot: Sync is disabled until the next reboot.
  get_sync_disabled_for_tests
      Prints one of the SYNC_DISABLED_MODE values, see set_sync_disabled_for_tests
```

</details>

---

&nbsp;





## Credits

A huge thanks to the OP @V-no-A who ran like a gazillion commands I kept sending him that made this write up possible. A huge thanks to @MishaalRahman as well for getting the news out by initiating the https://www.xda-developers.com/android-12-background-app-limitations-major-headache article and @Incipiens (AdamConwayIE) for testing and writing it up and for the credit as well.
