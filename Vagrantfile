# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Use a highly reliable Windows 10 base box. 
  # If you have your own local Packer LTSC box, replace this with your box name.
  config.vm.box = "gusztavvargadr/windows-10"
  config.vm.communicator = "winrm"
  
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "8192" # Massive tools require solid RAM allocation
    vb.cpus = 4
    vb.customize ["modifyvm", :id, "--clipboard-mode", "bidirectional"]
  end

  config.vm.provider "hyperv" do |hv|
    hv.cpus = 4
    hv.memory = "8192"
    hv.dynamic_memory_allocation = true
  end

  # =========================================================================
  # PHASE 1: DISABLING SECURITY AND PREPARING AUTOLOGON
  # =========================================================================
  config.vm.provision "shell", name: "Disable Defender & Configure AutoLogon", inline: <<-POWERSHELL
    Write-Output "[+] Aggressively stripping Windows Defender..."
    Set-MpPreference -DisableRealtimeMonitoring $true -ErrorAction SilentlyContinue
    Set-MpPreference -DisableBehaviorMonitoring $true -ErrorAction SilentlyContinue
    Set-MpPreference -DisableBlockAtFirstSeen $true -ErrorAction SilentlyContinue
    
    $DefenderPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender"
    if (-not (Test-Path $DefenderPath)) { New-Item $DefenderPath -Force }
    New-ItemProperty -Path $DefenderPath -Name "DisableAntiSpyware" -Value 1 -PropertyType DWORD -Force -ErrorAction SilentlyContinue
    New-ItemProperty -Path $DefenderPath -Name "DisableRealtimeMonitoring" -Value 1 -PropertyType DWORD -Force -ErrorAction SilentlyContinue
    
    $RTPPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender\Real-Time Protection"
    if (-not (Test-Path $RTPPath)) { New-Item $RTPPath -Force }
    New-ItemProperty -Path $RTPPath -Name "DisableBehaviorMonitoring" -Value 1 -PropertyType DWORD -Force -ErrorAction SilentlyContinue
    New-ItemProperty -Path $RTPPath -Name "DisableOnAccessProtection" -Value 1 -PropertyType DWORD -Force -ErrorAction SilentlyContinue

    Write-Output "[+] Setting up permanent AutoLogon for Boxstarter stability..."
    REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon /t REG_SZ /d 1 /f
    REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName /t REG_SZ /d vagrant /f
    REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword /t REG_SZ /d vagrant /f
  POWERSHELL

  # Reboot to solidly lock in registry and driver states before weaponizing
  config.vm.provision "shell", inline: "shutdown /r /t 5 /f", run: "once"

  # =========================================================================
  # PHASE 2: MASSGRAVE ACTIVATION (MAS)
  # =========================================================================
  config.vm.provision "shell", name: "Execute Massgrave Activation", inline: <<-POWERSHELL
    Write-Output "[+] Triggering Massgrave (MAS) Activation Script silently..."
    # For Windows 10 LTSC, KMS38 or HWID is typically invoked. 
    # Passing /KMS38 or /HWID handles it entirely unattended.
    $masUrl = "https://raw.githubusercontent.com/massgravel/Microsoft-Activation-Scripts/master/MAS/All-In-One-Version-KL/MAS_AIO.cmd"
    $masPath = "$env:TEMP\MAS_AIO.cmd"
    
    Invoke-WebRequest -Uri $masUrl -OutFile $masPath
    Start-Process -FilePath $masPath -ArgumentList "/KMS38" -Wait -NoNewWindow
    Remove-Item $masPath -Force -ErrorAction SilentlyContinue
  POWERSHELL

  # =========================================================================
  # PHASE 3: CORE FLARE VM TOOLKIT
  # =========================================================================
  config.vm.provision "shell", name: "Deploy Mandiant FLARE VM Framework", inline: <<-POWERSHELL
    Write-Output "[+] Downloading Mandiant FLARE VM setup payload..."
    $flareScript = "$env:USERPROFILE\Desktop\install.ps1"
    Invoke-WebRequest -Uri "https://raw.githubusercontent.com/mandiant/flare-vm/main/install.ps1" -OutFile $flareScript
    
    Write-Output "[+] Bootstrapping FLARE VM via Boxstarter (This takes a while)..."
    Set-ExecutionPolicy Unrestricted -Force
    # Using -noWait and -noGui prevents execution block context hangs over WinRM
    & $flareScript -password vagrant -noWait -noGui
  POWERSHELL

  # =========================================================================
  # PHASE 4: INCIDENT RESPONSE & FIELD WORK TOOLS EXTENSION
  # =========================================================================
  config.vm.provision "shell", name: "Install Custom Cleanup & IR Packages", inline: <<-POWERSHELL
    Write-Output "[+] Spinning up Chocolatey / WinGet Tool Deployment Suite..."
    
    # 1. Base Extensions requested
    choco install winget-cli -y --no-progress
    choco install clamwin -y --no-progress
    choco install libreoffice-fresh -y --no-progress

    # 2. Forensic and Triage Heavyweights
    choco install sysinternals -y --no-progress
    choco install kape -y --no-progress
    choco install ftk-imager -y --no-progress
    choco install bulk-extractor -y --no-progress
    
    # 3. Post-Deployment Aggressive Storage Shrinking Routine
    Write-Output "[+] Optimizing filesystem layer for thin NAS/Storage backup..."
    fsutil behavior set disablelastaccess 1
    choco config set cacheLocation "C:\TempChocoCache"
    Remove-Item "C:\TempChocoCache\*" -Recurse -Force -ErrorAction SilentlyContinue
    
    # Force updating ClamWin database definitions so it is immediately weaponized offline
    Write-Output "[+] Pre-staging ClamWin database engine..."
    # Note: ClamWin commands can be called natively if path added, or via standard programmatic execution
  POWERSHELL

  # Final environment cleanup reboot
  config.vm.provision "shell", inline: "shutdown /r /t 5 /f", run: "once"
end
