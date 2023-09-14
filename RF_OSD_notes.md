

I don't have docker compose, but I do have `docker`. So I ran the following to bring up the rotorflight container: 

```bash
docker build -t rotorflight_docker . # Build the container image 
docker run -it -v "/$(pwd):/rotorflight" --name rotorflight --rm rotorflight_docker /bin/bash
```

Then configured the `arm` build environment (I'm guessing)

```
root@45d102a27445:/rotorflight# make arm_sdk_install
```

Commented out the `#undef USE_OSD` and `#undef use_MAX7456` lines in [STM32_UNIFIED/target.h](https://github.com/rotorflight/rotorflight-firmware/blob/master/src/main/target/STM32_UNIFIED/target.h) (`/src/main/target/STM32_UNIFIED/target.h)`. I'm not strictly sure commenting out `#undef use_MAX7456` is necessary for digital, but I guess I'm erring on the side of caution here. 

And compiled the `STM32F7X2` target (presuming this is appropriate for my FC running a `STM32F722`):

```
root@45d102a27445:/rotorflight# make STM32F7X2
```

Then when it's flashed and dump all is loaded, do this biz from [Oscar Liang's Walksnail setup](https://oscarliang.com/setup-avatar-fpv-system/#unlocking-fcc-mode)

```
set osd_displayport_device = MSP
set displayport_msp_serial = x (should be 1 less than the UART number, e.g. if UART1, enter 0 here)
set vcd_video_system = PAL
save
```

## Simpler way of dealing with OSD 
Since the changes are so minimal, maintaining a fork is kind of ridiculous. Just pull the master repo, make the changes, and recompile. 

```
git checkout release/4.2.13 -b release_latest_w_OSD
```

- Comment out the `#undef USE_OSD` and `#undef use_MAX7456` `/src/main/target/STM32_UNIFIED/target.`.
- Start the docker container (make sure you're in the `rotorflight-firmware` directory)
    ```bash
    docker run -it -v "/$(pwd):/rotorflight" --name rotorflight --rm rotorflight_docker /bin/bash
    ```
- Recompile 
  - (If necessary)
        ```
        root@45d102a27445:/rotorflight# make arm_sdk_install
        ```
  - Build 
        ```
        root@45d102a27445:/rotorflight# make STM32F7X2
        ```
  - Flash 

- Syncing: 
  - Simply do a `git merge release/<latest>` (while in the `release_latest_w_OSD branch`). 
  - Double-check that the modifications to `target.h` are still there, which they should be. Everything else should get updated to the latest release. 

## Looks like this is gonna be a lot harder than it looked
Naturally. 

Get the release tag, merge it with my fork's main and fpv branch (git fetch remote-url "refs/tags/*:refs/tags/*")

`git fetch remote-url "refs/tags/*:refs/tags/*"`

Putting the latest release into `release_latest`. That should always be whatever the most recent release is, and serve as the effective master. The FPV branch should pull from that, for the moment. 

So now we can mess with the `rotorflight-firmware-fpv` branch until it works. If we can get it to work. 

### Pull requests 
On the betaflight discord, `haslinghuis` suggested I look up pull requests related to [MSP/VTX changes](https://github.com/betaflight/betaflight/pulls?q=vtx+msp). 

- It looks like [#11705 (VTX Device over MSP)](https://github.com/betaflight/betaflight/pull/11705) might be the place to start. 
- This would be followed by [#11913](https://github.com/betaflight/betaflight/pull/11913)  (which replaces `displayport_msp_serial`, and may not strictly be necessary), 
- and then [#11950](https://github.com/betaflight/betaflight/pull/11950), which may only be relevant to HDZero, I'm not sure, it may not be necessary, though.


### List of Changes ([#11705](https://github.com/betaflight/betaflight/pull/11705/files/4ca3b9428f759d6637f7982cba7722ff4135b496))

If this fails, try just copying these files over directly? 
- [ ] `docs/Serial.md` M 
- [28 ??] `make/source.mk` M 
- [ ] `src`
  - [ ] `main`
    - [ ] `build`
      - [26 ??] `debug.c` M
      - [27 ??] `debug.h` M
    - [ ] `cli`
      - [25] `settings.c` M 
    - [ ] `cms`
      - [0] `cms_menu_main.c` M
      - [1] `cms_menu_vtx_common.c` M 
      - [2] `cms_menu_vtx_msp.c` +
      - [3] `cms_menu_vtx_msp.h` +
    - [?] `drivers`
      - [4] `vtx_common.h` M 
    - [ ] `fc`
      - [5] `init.c` M 
      - [6] `tasks.c` M 
    - [ ] `io`
      - [7] `displayport_msp.c` M
      - [8] `serial.c` M
      - [9] `serial.h` M
      - [9] `vtx_msp.c` +
      - [10] `vtx_msp.h` +
    - [ ] `msp`
      - [11 ??] `msp.c` M
      - [12] `msp_box.c` M
      - [13] `msp_serial.c` M
      - [14] `msp_serial.h` M 
    - [ ] `pg`
      - [15] `msp.c` + 
      - [16] `msp.h` + 
      - [17] `pg_ds.h` M 
    - [ ] `sensors`
      - [18] `current.c` M 
    - [ ] `target`
      - [ ] `SITL`
        - [19] `target.h` M
      - [20] `common_post.h` M 
      - [21] `common_pre.h` M 
    - [ ] `telemetry`
      - [22] `telemetry.c` M 
    - [ ] `test`
      - [23] `Makefile` M 
      - [ ] `unit`
        - [24] `vtx_msp_unittest.cc` + 

- So this didn't work, which isn't at all surprising. I talked to some Betaflight people, namely `Belrik`? Gonna try a slightly simpler approach of just enabling some more stuff in the target. 
- Specifically, in `target.h`, comment out 
    ```c
    #undef  USE_OSD
    #undef  USE_CMS
    #undef  USE_MAX7456
    #undef  USE_DASHBOARD
    #undef  USE_RCDEVICE
    #undef  USE_VTX_CONTROL
    ```
- Aaand we'll try again. 
- This worked, which is really cool. Does it need all the changes? Or would this work in RotorFlight master? 
- Cool works fine with the master firmware all the same. 

Okay so I'm making a branch that is up to date with the latest RF release, but with those undefs commented out in target.h (`release_latest_w_OSD`). 

Compiled into `rotorflight_4.2.13_STM32F7X2_release_latest_w_OSD.hex` (changed the name of the output file). 

#### 

- Flash this firmware 
- Apply BETAFLIGHT defaults/target (RUSH-BLADE_F7_HD_betaflight.config)
- -> Battery voltage is present. 
- Apply [My Rotorflight mapping](https://docs.google.com/spreadsheets/d/1GOPNIWBpl4pt3RZKh50ooZDOG34uL7X_9s8GLVSmO5M/edit#gid=153958966) `Blade_Rotorflight_Mapping.txt`
- -> Battery voltage *still* present. 
- `set osd_displayport_device = MSP` (no effect)
- 
```
set osd_displayport_device = AUTO
set vcd_video_system = NTSC
```

**Try**
- [ ] Flash this (FPV) firmware 
- [ ] Apply MY RotorFlight target (`My_Blade_Rotorflight_target.txt`)
- [ ] Battery voltage is present?
- [ ] Run commands
    ```
    set osd_displayport_device = AUTO
    set vcd_video_system = NTSC
    ```
- [ ] Update DJI Goggles and Air Unit firmware. 


**Try**
- [x] Flash the **REGULAR**  Blade firmware 
- [x] Apply custom defaults 
- [x] Battery voltage is present? (Yes)
- [ ] ~~Apply MY RotorFlight target (`My_Blade_Rotorflight_target.txt`)~~
- [ ] ~~Battery voltage is present?~~
- [x] Apply [MY RotorFlight Mapping](https://docs.google.com/spreadsheets/d/1GOPNIWBpl4pt3RZKh50ooZDOG34uL7X_9s8GLVSmO5M/edit#gid=153958966) (`Blade_Rotorflight_Mapping.txt`)
- [x] Battery voltage is present? (Yes)
- [ ] **What the fuck turning on the receiver breaks the MSP connection to the DJI goggles.**
  - Wait no now it's working. 
  - Omg I'm such a fucking idiot. 
  - You need to use the onboard ADC to measure the battery voltage, OR have the motor plugged in and it set to ESC sensor (which I don't, at the moment). Also, it may just not work at all with the ESC sensor method. 
- [ ] Run commands
    ```
    set osd_displayport_device = AUTO
    set vcd_video_system = NTSC
    ```
- [ ] Update DJI Goggles and Air Unit firmware. 

