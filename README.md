simple-kdump
============

About
-----

This is a very simple kdump setup for Archlinux.

It has one and one function only, save the vmcore to `/var/crash/` of the root
fs.

There are only two dependency, `systemd` and `makedumpfile`.

HOW TO USE
----------

Install from [AUR](https://aur.archlinux.org/packages/simple-kdump)

- Modify `simple-kdump.conf`
  Note the kernel doesn't need to match the kernel you want to debug.
  So choose your favorite/smallest/whatever kernel and its initramfs.

  Some kernel configs must be enabled, fortunately `linux` and `linux-lts` all
  have the options enabled for x86_64 Archlinux.

  For ArchlinuxARM `linux-aarch64` kernels, `CONFIG_CRASH_DUMP=y` and
  `CONFIG_PROC_VMCORE=y` options are needed, and the PR is already submitted.

  And just use the regular boot option without `crashkernel=` option.

- Add `crashkernel=` option to boot entry and reboot
  Normally I'd recommend `512M` as I have hit cases where 256M is not large enough to
  decompress the huge 50M kdump specific initramfs generated by Fedora's dracut.

- Start and enable `simple-kdump-setup.service`

- Test if the setup works
  By `sync; echo c > /proc/sysrq-trigger` with root privilege.

  The system should reboot into an emergency shell, with `makedumpfile` running at the
  background:

```
[  OK  ] Started Emergency Shell.
[  OK  ] Reached target Emergency Mode.
[  OK  ] Started Collect the vmcore for simple-kdump.
```

  There will be a prompt for the emergency shell login, but you can/should ignore that.

  After the vmcore is collected, the system will reboot back to your regular setup.
  And `/var/crash/` will contain the last collected vmcore file.


IMPLEMENTATION
--------------

Unlike other distros, Archlinux doesn't have an easy to use way to setup kdump
to collect vmcore from crashed kernels.
There are projects like [kdumpst](https://gitlab.freedesktop.org/gpiccoli/kdumpst)
backed by Valve to improve the situation.

But unfortunately I hate GRUB2 and have too many custom kernels that GRUB2 can
never properly detect.

The freedom to choose different kernels and bootloaders is exactly why I'm using
Archlinux, and also why it's so hard to implement kdump.

So I just come up with the super simple setup.

The core of the simplicity comes from letting user to choose whatever kernel/initramfs
combination as the kexec kernel/initramfs, not reusing the current kernel/initramfs,
as long as the combination can boot and have needed options enabled.

This avoids two complex workload:

- Detect the booting kernel/initramfs/cmdline
- Rebuild a dedicated kdump initramfs

Instead use `systemd.unit=` option to let the kexec environment to boot into a special
emergency target which also collects the vmcore and reboot.

But this also means there is a limitation.

If the crash caused serious filesystem corruption that your local fs can not be mounted,
the setup will not work at all.

Fortunately even during my filesystem development work, I haven't really hit such case.
