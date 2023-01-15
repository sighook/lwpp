# Linux wifi pentest patches

A collection of patches for vanilla linux kernel,
useful for pentesters and security engineers.

Supported branches:

- 5.3 (~~EOL: December 2019~~)

- 5.4 (EOL: December 2025)

## Setup

- Apply the patches that correspond to your kernel branch. For example:

  Note that you may use `--dry-run` option to check if there is no errors before applying.

  ```sh
  _BRANCH=5.4
  cd /usr/src/linux${_BRANCH}
  for _PATCH in ../linux-wifi-pentest-patches/${_BRANCH}/*.patch; do
      patch -p1 -i ${_PATCH}
  done
  ```

- Build the kernel & modules. For example:

  ```sh
  cp /boot/config .config
  make olddefconfig
  make -j$(nproc) all
  make modules_install
  cp arch/x86/boot/bzImage /boot/vmlinuz
  cp .config /boot/config
  ```

- Update bootloader. For example (in case you're using GRUB):

  ```sh
  grub-mkconfig -o /boot/grub/grub.cfg
  ```

- Reboot.

- Have fun!

## Description

- `0001-fix-QoS-overwriting.patch`

  When injecting packets the Quality of Service (QoS) header was
  being overwritten by the driver.

  This patch tells to the driver to not overwrite the QoS header
  when the device is in the monitor mode.

  Thanks to [Mathy Vanhoef](http://www.mathyvanhoef.com/2012/09/compat-wireless-injection-patch-for.html).

- `0002-fix-the-channel-changing-of-the-monitor-interface.patch`

  We can't change the channel when "normal" virtual interfaces are also using the device.

  So, it's unable to change the channel of the monitor interface with the error message
  "SET failed on device mon0 ; Device or resource busy."

  Practically this means that (if you don't apply this patch) we
  need to disable them by executing "ifconfig wlanX down" until we
  only have monitor interfaces over.

  However disabling them all the time is annoying and most of the
  time if you're playing with monitor mode you're not using the
  device in a normal mode anyway.

  Thanks to [Mathy Vanhoef](http://www.mathyvanhoef.com/2012/09/compat-wireless-injection-patch-for.html).

- `0003-fix-injecting-fragments-on-rtl8187-based-wifi-cards.patch`

  Injecting fragments was not working properly on rtl8187: only the
  the first fragment was being transmitted.

  A simple test to further isolate the issue by instructing to driver
  to send the following frames (from a userland program):

  *  Send the first fragment
  *  Send an ARP request packet
  *  Send the second fragment, which is the last one

  It turned out the device actually transmits the ARP request packet first, and only then sends the first fragment!

  It first waiting for ALL the fragments before it begins sending them.

  Furthermore, it would only send the next fragment once the previous one has been acknowledged (which isn't detected in monitor mode, hence only the first fragment is transmitted).

  Thanks to [Mathy Vanhoef](http://www.mathyvanhoef.com/2012/09/compat-wireless-injection-patch-for.html).

- `0004-skip-frame-ACKing-renumbering-handle-sequence-by-use.patch`

  Packet injection may want to control the sequence number, so if an injected packet is found, skip renumbering it.

  Also make the packet NO_ACK to avoid excessive retries (ACKing and retrying should be handled by the injecting application).

  **FIXME** This may break hostapd and some other injectors. This should be done using a radiotap flag.

  Thanks to [aircrack patches](http://patches.aircrack-ng.org/). *(Original author is unknown.)*

- `0005-Enable-monitoring-and-injection-for-the-Zydax-1211rw.patch`

  _Enable monitoring and injection for the Zydax 1211rw driver_

  Thanks to [aircrack patches](http://patches.aircrack-ng.org/). *(Original author is unknown.)*

- `0006-Override-regdomain-hardcoded-in-EEPROM-with-custom-v.patch`

  _Override regdomain hardcoded in EEPROM with custom value_

  Usage:

  Get your country code from `linux/drivers/net/wireless/ath/regd.h` and supply as a parameter.

  Example:

  `sudo modprobe carl9170 override_eeprom_regdomain=<CODE>`

  Thanks to [Paul Fertser](fercerpav@gmail.com).

- `0007-carl9170-Enable-sniffer-mode-promisc-flag-to-fix-inj.patch`

  _carl9170: Enable sniffer mode promisc flag to fix injection_

  The removal of the `AR9170_MAC_SNIFFER_ENABLE_PROMISC` flag to fix an issue
  many years ago caused the AR9170 to not be able to pass probe response
  packets with different MAC addresses back up to the driver. In general
  operation, this doesn't matter, but in the case of packet injection with
  `aireplay-ng` it is important. aireplay-ng specifically injects packets with
  spoofed MAC addresses on the probe requests and looks for probe responses
  back to those addresses. No other combination of filter flags seem to fix
  this issue and so `AR9170_MAC_SNIFFER_ENABLE` is required to get these packets.

  This was originally caused by commit `e0509d3bdd7365d06c9bf570bf9f11` which
  removed this flag in order to avoid spurious ack noise from the hardware.

  In testing for this issue, keeping this flag but not restoring the
  `AR9170_MAC_RX_CTRL_ACK_IN_SNIFFER` flag on the `rc_ctrl` seems to solve this
  issue, at least with the most current firmware v1.9.9.

  Thanks to [Steve deRosier](derosier@cal-sierra.com)

- `0008-mac80211-ignore-AP-power-level-when-tx-power-type-is.patch`

  _mac80211: ignore AP power level when tx power type is "fixed"_

  In some cases a user might want to connect to a far away access point,
  which announces a low tx power limit. Using the AP's power limit can
  make the connection significantly more unstable or even impossible, and
  mac80211 currently provides no way to disable this behavior.

  To fix this, use the currently unused distinction between limited and
  fixed tx power to decide whether a remote AP's power limit should be
  accepted.

  Usage:

  `iw dev XXX set txpower fixed YYY`

  Thanks to [Felix Fietkau](nbd@openwrt.org)

- `0009-ath9k-override-regdomain-hardcoded-in-EEPROM-with-cu.patch`

  _ath9k: override regdomain hardcoded in EEPROM with custom value_

  Get your country code from linux/drivers/net/wireless/ath/regd.h
  and supply as a parameter:

  `modprobe ath9k_hw override_eeprom_regdomain=<CODE>; modprobe ath9k`

<!-- vim:ft=markdown
End of file. -->
