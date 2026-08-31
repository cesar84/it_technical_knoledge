# How to Recover From a Desktop Freeze (Acer Nitro ANV15-51 / Fedora 44)

**Symptom:** screen frozen, fan running, no response from keyboard, mouse, or
pointer. Root cause on this machine: GNOME automatic suspend aborts on the
USB4/Thunderbolt controller and wedges the NVIDIA driver + the Wayland
compositor (`gnome-shell`). The kernel is usually still alive underneath.

Try the steps below **in order** — from least to most destructive. Only use a
hard power-off as the last resort.

---

## 0. One-time preparation (do this now, while the system works)

These make recovery possible later.

### Enable full Magic SysRq
Currently `kernel.sysrq = 16` (only `sync` allowed), so the REISUB sequence
does nothing. Enable it:

```bash
echo 'kernel.sysrq=1' | sudo tee /etc/sysctl.d/99-sysrq.conf
sudo sysctl --system
cat /proc/sys/kernel/sysrq   # should now print 1
```

### Enable SSH access from another device
So you can log in remotely when the screen is frozen:

```bash
sudo systemctl enable --now sshd
ip a        # note the LAN IP, e.g. 192.168.1.x
```

### (Recommended) Stop the trigger entirely
In the graphical session as user `simone`:

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

Or: Settings -> Power -> "Automatic Suspend" -> Off.

---

## 1. Switch to a text console

Press:

```
Ctrl + Alt + F3
```

(add **Fn** if the F-keys are set to multimedia mode: `Fn + Ctrl + Alt + F3`)

- If a text login prompt appears, the kernel and login stack are fine — go to
  **Step 2**.
- If nothing happens (still frozen), skip to **Step 4**.

### Return to the graphical desktop
The GUI session runs on **tty2**, so:

```
Ctrl + Alt + F2
```

If tty2 is black, try `Ctrl + Alt + F1` (GDM login screen).

---

## 2. From the text console: restart just the graphics stack

Log in as `simone`, then:

```bash
sudo systemctl restart gdm
```

This kills the frozen session (you lose unsaved work in it) but does **not**
reboot the machine. You land back at the GDM login screen. Log in normally.

If that hangs, try a full clean reboot instead:

```bash
sudo systemctl reboot
```

---

## 3. From another computer: SSH in

```bash
ssh simone@192.168.1.x
```

Then the same options:

```bash
sudo systemctl restart gdm      # restart graphics only
sudo systemctl reboot           # clean full reboot
```

Check what happened while you are in there:

```bash
journalctl -b -0 -e
journalctl -b -0 | grep -Ei 'suspend|usb usb4|unresponsive|Mutter|gnome-shell'
```

---

## 4. Magic SysRq: forced but orderly reboot (REISUB)

Use this when the console does not respond and SSH is not available.
Requires Step 0 (`kernel.sysrq=1`) to have been done.

Hold **Alt + SysRq** (SysRq = the **PrtSc / Stamp** key; add **Fn** if needed)
and, keeping them held, tap these keys slowly, ~1 second apart:

```
R  E  I  S  U  B
```

| Key | Action |
|-----|--------|
| R | take keyboard back from the frozen display server |
| E | send SIGTERM to all processes |
| I | send SIGKILL to all processes |
| S | flush (sync) all filesystems to disk |
| U | remount all filesystems read-only |
| B | reboot immediately |

Mnemonic: **R**aising **E**lephants **I**s **S**o **U**tterly **B**oring.

Wait ~2-3 seconds between **S** and **U** so the disk sync can finish.

> Quick variant: `Alt + SysRq + B` alone reboots instantly with **no** sync —
> almost as unsafe as the reset button. Prefer the full REISUB.

---

## 5. Last resort: hard power-off

Only if everything above failed.

- Hold the **power button ~10 seconds** until it powers off, then power on.
- This is an unclean shutdown: possible filesystem check on next boot, and any
  unsaved data is lost. Btrfs (this system's root) tolerates it well but it is
  still not free.

---

## After you are back in

Confirm the cause and check for damage:

```bash
# what the previous (crashed) boot logged near the freeze
journalctl -b -1 -e
journalctl -b -1 | grep -Ei 'suspend|usb usb4|failed to suspend|unresponsive|Mutter'

# filesystem health
sudo btrfs device stats /
sudo dmesg | grep -Ei 'error|fail|btrfs'
```

If you see `usb usb4: PM: failed to suspend async: error -16` again, the
suspend bug is still active — keep automatic suspend disabled (Step 0) until an
Acer BIOS update fixes it.
