# Automating Windows Installation with Autounattend.xml and UUP dump


# Automating Windows Installation with Autounattend.xml and UUP dump

Automating a Windows unattended installation can save hours of repetitive work — whether for lab testing, VM deployment, or mass corporate rollout. By using an `Autounattend.xml` file, you predefine installation steps such as language, partitioning, product key, and even post-install configuration, all without user input.

In this guide, you'll learn:

1. What `Autounattend.xml` is and how it works
2. How to create it
3. How and where to place it
4. How to integrate language packs
5. How to use **UUP dump** for building custom ISOs
6. Example: fully automated workflow
7. Troubleshooting common issues

---

## 1. Understanding Autounattend.xml

`Autounattend.xml` is a Windows **answer file** that automates setup by supplying preconfigured values.

**Phases (passes) it controls:**

- **windowsPE** — disk partitioning, language, TPM/Secure Boot bypass
- **specialize** — drivers, hostname, networking
- **oobeSystem** — accounts, privacy, first-run experience

**Common settings include:**

- Language and locale
- Disk layout and formatting
- Edition selection (from `install.wim`/`install.esd`)
- Product keys
- Driver injection
- Registry tweaks and PowerShell scripts

When Windows Setup detects `Autounattend.xml`, it follows the specified configuration automatically.

---

## 2. Creating Autounattend.xml

### **Option 1: Windows SIM (System Image Manager)**

- Part of **Windows ADK**
- Loads an `install.wim` and provides a GUI for configuring settings
- Validates before saving
- **Download:** [Windows ADK](https://learn.microsoft.com/windows-hardware/get-started/adk-install)

### **Option 2: Online Generators**

- Example: [Schneegans Generator](https://schneegans.de/windows/unattend-generator/)
- Offers quick form-based setup for:
  - Language
  - Disk layout
  - User accounts
  - TPM/Secure Boot bypass
- Produces ready-to-use XML

### **Option 3: Manual Editing**

- Edit with VS Code, Notepad++, etc.
- Follow the [Unattended Setup Reference](https://learn.microsoft.com/windows-hardware/customize/desktop/unattend/)
- Always validate with Windows SIM

---

## 3. Placing Autounattend.xml

Windows Setup searches multiple locations:

### **A. Root of USB**

- Plug in a USB with only `Autounattend.xml`
- Works with official ISOs
- Requires 2 devices

### **B. Integrated into ISO**

- Mount ISO → Copy contents → Place XML in root → Rebuild ISO
- **Windows (ADK):**
  ```powershell
  oscdimg -m -o -u2 -udfver102 C:\WinISO C:\Win11-Auto.iso
  ```
- **Linux:**
  ```bash
  mkisofs -b boot/etfsboot.com -no-emul-boot -boot-load-size 8 -o Win11-Auto.iso WinISO/
  ```
*Tip:* use `oscdimg` on Windows, `mkisofs` on Linux.

### **C. Second Virtual CD/DVD**

- Attach small ISO with `Autounattend.xml` as secondary media in VM
- Avoids editing original ISO

### **D. $OEM$ Structure**

- Place in `sources\$OEM$\$1`
- Common in OEM factory setups

**Note:** Windows scans all connected drives, prioritizing removable media.

---

## 4. Integrating Language Packs

1. Download `.cab` packages:
   - `Microsoft-Windows-Client-Language-Pack_x64_<lang>.cab`
   - `Microsoft-Windows-LanguageFeatures-Basic-<lang>-Package.cab`
2. Inject with DISM:
   ```powershell
   dism /Mount-Wim /WimFile:install.wim /Index:1 /MountDir:mount
   dism /Image:mount /Add-Package /PackagePath:lp.cab
   dism /Unmount-Wim /MountDir:mount /Commit
   ```
3. Update `Autounattend.xml` `<UILanguage>`, `<SystemLocale>`, `<InputLocale>`

*Tip: If you use UUP dump and select the desired language during build, you can skip manual integration.*

---

## 5. Using UUP dump

[**UUP dump**](https://uupdump.net) — a tool to download official Microsoft Windows builds from UUP servers with custom options.

### Overview

- Any Windows version (11, 10, Server)
- Edition & language selection
- Language packs & Features on Demand
- Always latest builds from Microsoft CDN

### Converter Scripts

- `convert-UUP.cmd` (Windows)
- `convert.sh` (Linux/macOS)

**Functions:**

- Assemble ISO from UUP files
- Integrate updates, drivers, language packs

Example (Linux):

```bash
sudo apt install aria2 cabextract wimtools chntpw genisoimage
./convert.sh wim
```

---

## 6. Automated Workflow Example

```
UUP dump → ISO Build → Add Autounattend.xml → (Optional) Add Language Packs → Deploy
```

1. **UUP dump** — Select build, edition, language → Download script
2. **Convert** — Run `convert.sh` or `convert-UUP.cmd` to get ISO
3. **Integrate XML** — Place in root of ISO or separate ISO
4. **Optional:** Inject language packs
5. **Deploy** — Boot and enjoy unattended setup

Example Linux one-liner:

```bash
./uup_download_linux.sh && cp Autounattend.xml iso-root/ && mkisofs -o Win11-Auto.iso iso-root/
```

---

## 7. Troubleshooting Common Errors

### XML Parse Error

- Validate in Windows SIM
- Check for unsupported elements

### File Not Found

- Ensure exact filename `Autounattend.xml`
- Place in root of media

### Driver Missing

- Inject via DISM before install

### Language Not Applied

- Match `<UILanguage>`, `<UserLocale>`, `<SystemLocale>` with installed pack

### TPM/Secure Boot Block

- Add registry bypass in `windowsPE`

### Endless Reboot

- Simplify XML
- Match partition layout to boot mode

**Logs:**

- `X:\Windows\Panther\setuperr.log`
- `X:\Windows\Panther\setupact.log`

---

## 8. Why Use This

Combining `Autounattend.xml` + UUP dump lets you:

- Build **fully automated** ISOs
- Preconfigure language, edition, updates
- Save time in repetitive installations
- Maintain consistent environments for VMs, labs, and bare-metal deployments



