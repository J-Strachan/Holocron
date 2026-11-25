---
isCurrent: true
needsUpdating: true
---

### Lab Overview & Summary

In this practical lab, you will perform core operating system administration and hardware integration tasks on **Ubuntu 26.04 LTS**. By working directly in the Linux terminal and the Arduino IDE, you will demonstrate three essential IT support skills:
* **Operating System Optimisation (3.1):** Measuring disk storage, removing unused package caches, and trimming old `systemd` logs to reclaim drive space.
* **CLI & GUI Customisation (3.2):** Using command-line tools (`~/.bashrc` and `gsettings`) to personalise both your terminal prompt/aliases and the GNOME desktop environment.
* **Driver Installation & Verification (3.3):** Identifying Linux kernel USB-to-serial drivers (`lsusb` and `dmesg`), configuring `dialout` port permissions, and uploading the **Blink** sketch to verify physical Arduino hardware operation.

> [!info] Assessment Instructions
> Complete each practical task on your **Ubuntu 26.04 LTS** workstation or virtual machine. Record your terminal outputs in the tables provided and capture clear, full-window screenshots (showing your username and prompt) for your digital assessment portfolio.

## Lab Workflow Overview

```mermaid
flowchart LR
    A["Part 1: Criterion 3.1<br/>Storage & Cache Cleanup<br/>(df, apt, journalctl)"] --> B["Part 2: Criterion 3.2<br/>CLI & GUI Customisation<br/>(.bashrc & gsettings)"]
    B --> C["Part 3: Criterion 3.3<br/>Arduino Driver & Blink Test<br/>(lsusb, dmesg, dialout)"]
    C --> D["Part 4: Submission<br/>Portfolio Checklist"]

    style A fill:#1e293b,stroke:#E95420,stroke-width:2px,color:#f8fafc
    style B fill:#1e293b,stroke:#77216F,stroke-width:2px,color:#f8fafc
    style C fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style D fill:#1e293b,stroke:#34d399,stroke-width:2px,color:#f8fafc
```

---

## Part 1: Optimise Operating System Using Tools & Utilities (Criterion 3.1)

Over time, cached package archives and old system logs consume valuable disk space. In this task, you will measure available storage, safely reclaim disk space using Ubuntu's package management and journal utilities, and verify the improvement.

### Steps

1. Open the Terminal application (press `Ctrl + Alt + T`).
2. Inspect your current root filesystem disk usage in human-readable format by running:
   ```bash
   df -h /
   ```
3. Record your **Before Cleanup** storage figures in the table below.
4. Remove orphaned software dependencies and clear cached `.deb` installation files from `/var/cache/apt/archives`:
   ```bash
   sudo apt autoremove -y && sudo apt clean
   ```
5. Trim archived `systemd` system logs older than 7 days to free additional space:
   ```bash
   sudo journalctl --vacuum-time=7d
   ```
6. Re-run `df -h /` and record your **After Cleanup** storage figures below.

### Storage Measurement Record

| Measurement Stage | Total Size | Used Space | Available Space |
| :--- | :--- | :--- | :--- |
| **Before Cleanup** (`df -h /`) | | | |
| **After Cleanup** (`df -h /`) | | | |

> [!example] Portfolio Screenshot Evidence 3.1
> Capture a terminal screenshot displaying your initial `df -h /` output, the execution of `apt autoremove`, `apt clean`, and `journalctl --vacuum-time=7d`, and your final `df -h /` output.

---

## Part 2: Customise GUI & CLI Using Command Line Techniques (Criterion 3.2)

System administrators routinely configure both the Command Line Interface (CLI) and Graphical User Interface (GUI) from the terminal rather than clicking through menus manually. Complete both sections below using terminal commands only.

### Step 2A: Customise the Bash Command Line Interface (`~/.bashrc`)

1. Create a backup copy of your default Bash configuration file before editing:
   ```bash
   cp ~/.bashrc ~/.bashrc.bak
   ```
2. Append a custom command alias (`sysinfo`) and a green/blue colourised prompt to your `~/.bashrc` file, then reload the shell configuration:
   ```bash
   echo "alias sysinfo='uname -sr && uptime -p && free -h'" >> ~/.bashrc
   echo 'export PS1="\[\e[1;32m\]\u@\h\[\e[0m\]:\[\e[1;34m\]\w\[\e[0m\]\$ "' >> ~/.bashrc
   source ~/.bashrc
   ```
3. Test your new alias by typing `sysinfo` in the terminal and pressing `Enter`.

### Step 2B: Customise the Ubuntu 26.04 GNOME Desktop via `gsettings`

Use the GNOME settings CLI utility (`gsettings`) to modify the desktop colour scheme, dock position, icon size, and top-bar clock directly from the terminal:

```bash
# Switch Ubuntu 26.04 desktop to Dark Mode
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'

# Move the Ubuntu Dock to the bottom and set icon size to 36
gsettings set org.gnome.shell.extensions.dash-to-dock dock-position 'BOTTOM'
gsettings set org.gnome.shell.extensions.dash-to-dock dash-max-icon-size 36

# Display weekday and seconds on the top bar clock
gsettings set org.gnome.desktop.interface clock-show-weekday true
gsettings set org.gnome.desktop.interface clock-show-seconds true
```

> [!example] Portfolio Screenshot Evidence 3.2
> Capture a screenshot showing both your terminal window (with the custom prompt, `sysinfo` output, and `gsettings` commands) and the updated Ubuntu 26.04 desktop (Dark Mode enabled, Dock at bottom, weekday/seconds visible on top bar).

---

## Part 3: Final Portfolio Submission Checklist

Tick each item below before submitting your completed lab portfolio for assessment:

- [ ] **Criterion 3.1:** Completed storage table and screenshot showing `df -h /` before and after running `apt autoremove`, `apt clean`, and `journalctl --vacuum-time=7d`.
- [ ] **Criterion 3.2:** Screenshot showing custom Bash prompt/alias (`~/.bashrc`) and GNOME desktop changes applied via `gsettings`.

---

### Summary & Key Takeaways

* **OS Maintenance (`apt` & `journalctl`):** Regularly clearing cached `.deb` archives (`apt clean`), unused dependencies (`apt autoremove`), and old system logs (`journalctl --vacuum-time`) keeps Ubuntu responsive and prevents storage bottlenecks.
* **Scriptable Configuration (`~/.bashrc` & `gsettings`):** Modifying shell profiles and desktop settings via CLI commands allows administrators to automate and reproduce consistent environments faster than manual GUI changes.