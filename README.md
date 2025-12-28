# Linux fan control on motherboards with ITE IT8665E (it87)

If your motherboard uses the **ITE IT8665E Super I/O** chip, Linux can often *read* fan speeds but not *control* them—until you expose PWM control through the `it87` driver.

This guide shows, step by step, how to:
- enable PWM fan control (`pwm1`, `pwm2`, …) under `/sys/class/hwmon/`
- map each `pwmX` to a real fan header (CHA_FAN / CPU_OPT / pump, etc.)
- then use a fan control app (like **CoolerControl**) to build proper curves (including GPU-temp-based curves which UEFI usually doesn't offer)

---

## What you’re trying to achieve

Linux fan control needs two things:

1) **A temperature to react to** (CPU, GPU, etc.)  
2) **A knob to turn** (PWM control for your fan headers)

On IT8665E systems, the “knob” is provided by the kernel module **`it87`**.  
Once it is working, you’ll see files like:

- `pwm1`, `pwm2`, … (fan power / duty)
- `pwm1_enable`, `pwm2_enable`, … (auto vs manual control)

When those files exist, you can control motherboard fan headers from Linux reliably.

---

## Warning

Some boards have an ACPI “resource conflict” with the Super I/O chip. In that case the driver may detect the chip but refuse to expose controls unless you pass:

- `ignore_resource_conflict=1`

This is common. It usually works fine on desktops, but if you ever get weird ACPI/suspend issues later, this setting is the first thing to revisit.

---

## 1) Install the basics

You need:
- `lm-sensors` (the `sensors` command + detection tools)
- optionally: `i2c-tools` (useful for debugging on some systems)
- optionally: your fan control app (e.g. CoolerControl)

### Arch
```bash
sudo pacman -S --needed lm_sensors i2c-tools
```

### Debian / Ubuntu / Linux Mint / Pop!_OS
```bash
sudo apt update
sudo apt install -y lm-sensors i2c-tools
```

### Fedora
```bash
sudo dnf install -y lm_sensors i2c-tools
```

### openSUSE (Leap / Tumbleweed)
```bash
sudo zypper install -y sensors i2c-tools
# (package name can be lm_sensors or sensors depending on the release)
```

### Others
If your distribution/package manager is not listed, search for `lm-sensors` (or `lm_sensors`) in your package manager.

---

After installing, detect sensors and confirm `sensors` works:
```bash
sudo sensors-detect --auto
sensors
```

---

## 2) Check: do you already have `it8665` and `pwm*`?

List hwmon devices:
```bash
for d in /sys/class/hwmon/hwmon*; do
  echo "$d: $(cat $d/name)"
done
```

If you see a device named `it8665`, grab its path:
```bash
d=$(for x in /sys/class/hwmon/hwmon*; do
  [ "$(cat $x/name)" = "it8665" ] && echo $x
done)
echo "it8665 hwmon path: $d"
```

Now check if PWM exists:
```bash
ls "$d" | grep -E '^pwm[0-9]+' | sort
```

If you see `pwm1`, `pwm2`, … you can skip to mapping (section 5).

If you *don’t* see `it8665` or `pwm*`, continue.

---

## 3) Enable PWM control: load `it87`

First try the normal way:
```bash
sudo modprobe -r it87 2>/dev/null || true
sudo modprobe it87
```

If that still doesn’t create `it8665`, use the common fix:
```bash
sudo modprobe -r it87 2>/dev/null || true
sudo modprobe it87 ignore_resource_conflict=1
```

Re-check `/sys/class/hwmon` again (same commands as above).  
You should now see `it8665` and `pwm*`.

---

## 4) Make it permanent (so it works after reboot)

```bash
echo "options it87 ignore_resource_conflict=1" | sudo tee /etc/modprobe.d/it87.conf
echo "it87" | sudo tee /etc/modules-load.d/it87.conf
```

Reboot, then confirm `it8665` still exists.

---

## 5) Don’t fight your own fan-control app (avoid EBUSY)

If you have CoolerControl running, it may “own” the PWM channels.  
When you try to write `pwmX` manually you can get:

- `Device or resource busy`

That’s not a bug - Linux is preventing two controllers from fighting.

Stop it while you do mapping/tests.

Example for coolercontrol:

```bash
sudo systemctl stop coolercontrold 2>/dev/null || true
sudo systemctl stop coolercontrol 2>/dev/null || true
sudo pkill coolercontrold 2>/dev/null || true
```

Confirm it’s stopped:
```bash
ps aux | grep -i coolercontrol | grep -v grep || true
```

---

## 6) Map `pwm1..pwmN` to real fan headers (the important part)

The system gives you “pwm1”, “pwm2”, … but you need to learn what they actually control.

### 6.1 Find the correct `it8665` hwmon path (again)
Hwmon numbers can change after reboot, so do:
```bash
d=$(for x in /sys/class/hwmon/hwmon*; do
  [ "$(cat $x/name)" = "it8665" ] && echo $x
done)
echo "it8665 hwmon path: $d"
```

### 6.2 Watch fan RPM in a readable way
On many ASUS boards, `asus_wmi_sensors` shows friendly names (CPU Fan, Chassis Fan 1/.../n, CPU OPT, ...).

In one terminal:
```bash
watch -n 1 'sensors | grep -E "Chassis Fan|CPU OPT|CPU Fan|Pump|AIO"'
```

### 6.3 Test one PWM channel
In another terminal (example for `pwm1`):

```bash
# put this channel into manual mode (usually 1 = manual)
echo 1 | sudo tee "$d/pwm1_enable"

# bump the PWM up, then down
echo 200 | sudo tee "$d/pwm1"
sleep 4
echo 120 | sudo tee "$d/pwm1"
sleep 4
```

Watch which fan RPM changes in the other terminal.
That fan header is controlled by `pwm1`.

Repeat for `pwm2`, `pwm3`, `pwm4` (and more if you need them).

> Tip: Start with 120–200.  

Write your mapping down, e.g.:
- `pwm1 → CPU_FAN/CPU_OPT`
- `pwm2 → CHA_FAN1`
- `pwm3 → CHA_FAN2`
- `pwm4 → CHA_FAN3`
- `pwm5/pwm6 → unused / pump / not connected`

---

## 7) Build fan curves (CoolerControl example)

Start CoolerControl again:
```bash
sudo systemctl start coolercontrold 2>/dev/null || sudo systemctl start coolercontrol
```

In the UI:
1) Scroll until you find it8665
2) Choose a fan
3) Create a control profile
4) Choose the correct temp source for the selected fan 
5) Create an optimal curve (needs some testing to tune it just right for the best performance and silence)

---

## Why I did this

BIOS fan control is too limited:
- fan curves are too simple,
- you can’t combine multiple temperature sources,
- you often can’t use **GPU temperature** as a control source.

- GPU fans were replaced with larger fans connected to motherboard headers
- the goal is **GPU temperature → those motherboard headers**
- Linux becomes quiet at idle and still cool under GPU load, with detailed curves

---

## One working setup (example, not guaranteed for your board)

Example board: ASUS ROG Strix B450‑F Gaming II  
Example mapping found by testing:
- `pwm1` → CPU_FAN/CPU_OPT  
- `pwm2` → Chassis Fan 1  
- `pwm3` → Chassis Fan 2  
- `pwm4` → Chassis Fan 3  

Your mapping may differ—always test and confirm.
