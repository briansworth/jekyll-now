---
layout: post
title: PowerShell - Automate SCCM PreReq Install
description: Script to install SCCM prerequisites
---

This is a part 2 to my original 
[post]({{ site.baseurl }}/PowerShell-SCCM-Offline-PreRequisites/)
about setting up SCCM on a server disconnected from the internet. 

<p>
  That being said, using this automation does not mean it will not 
  speed up your SCCM deployment on an internet connected server. 
  There is nothing stopping you from running the 
  download script on your SCCM server, 
  then running this install script using the downloaded files directly. 
  If you have to build SCCM environments often, 
  storing and keeping the downloaded files in a zip file would make the 
  install process much faster.  
  All you need then is to copy the files and run the install script.
</p>

<p>
  I should mention that the ActiveDirectory prerequisites 
  (creating the System Management container and extending the Schema) 
  are not included in this script.  
  I might cover that in a future post.
</p>

### What the Script Will Do
----

<p>
  The script is going to install all the required Windows Features for SCCM. 
  It will install the required components from the ADK. 
  Since SCCM requires .NetFramework 3.5, and that component is removed from 
  Windows, you will need to mount the Windows ISO to the computer to install 
  that specific feature.  
  You will need to specify the Windows source path in the script to get that 
  feature installed. 
  The script will skip this feature if you have not specified that parameter.
</p>

### The Script
----

<p>
  Examples for how to use it are below the code. 
  It is recommended to copy this code to a text editor, 
  then save it as a ps1 file (SccmPreReqInstall.ps1 for example). 
  You can then run it in an elevated PowerShell session.
</p>

```powershell
[CmdletBinding()]
Param(
  [Parameter(Mandatory=$true,Position=0)]
  [String]$prereqSourcePath,

  [Parameter(Position=1)]
  [ValidateScript({Test-Path -Path $_})]
  [String]$windowsMediaSourcePath
)

Function isAdministrator {
  $wid=[Security.Principal.WindowsIdentity]::GetCurrent()
  $principal=New-Object -TypeName Security.Principal.WindowsPrincipal `
    -ArgumentList $wid
  $adminRole=[Security.Principal.WindowsBuiltInRole]::Administrator

  return $principal.IsInRole($adminRole)
}

Function GetADKInstalledFeatures {
  [CmdletBinding()]
  Param()

  $kitsRegPath='HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows Kits'

  Try{
    $winKitsInstall=Test-Path -Path $kitsRegpath
    if(!$winKitsInstall){
      Write-Error "[$kitsRegPath] does not exist. ADK not installed." `
        -ErrorAction Stop
    }
    $installRoots=Get-ChildItem -Path "$kitsRegPath\Installed Roots" `
      -ErrorAction Stop

    if(!$installRoots){
      Write-Error "Install Roots key not found.  ADK features not installed." `
        -ErrorAction Stop
    }
    $installOptPath="$($installRoots[0].PSPath)\Installed Options"
    $installOpt=Get-Item -Path $installOptPath -ErrorAction Stop
    
    $installOptHash=New-Object -TypeName hashtable
    foreach($option in $installOpt.Property){
      $installOptHash.Add($option,$installOpt.GetValue($option))
    }
    Write-Output $installOptHash
  }Catch{
    Write-Error $_
  }
}

Function WaitForProcessEnd {
  [CmdletBinding()]
  Param(
    [String]$processName,

    [String]$msg
  )
  $processRunning=$true
  Write-Host $msg -NoNewline
  Do{
    Write-Host '.' -NoNewline
    Start-Sleep -Seconds 30
    $process=Get-Process -Name $processName `
      -ErrorAction SilentlyContinue
    if(!$process){
      $processRunning=$false
    }
  }While($processRunning)
}

Try{
  Write-Verbose "Admin?"
  $isAdmin=isAdministrator
  if(!$isAdmin){
    Write-Error "Not running as Admin. Run with Administrative permissions" `
      -ErrorAction Stop
  }

  Write-Verbose "Validating PreReq files"

  $prereqTest=Test-Path -Path $prereqSourcePath
  if(!$prereqTest){
    Write-Error "PreReq Source Path is invalid" `
      -ErrorAction Stop
  }

  $sccmPreReqPath=Join-Path -Path $prereqSourcePath -ChildPath prereq
  $adkPath=Join-Path -Path $prereqSourcePath -ChildPath adk
  $adkPePath=Join-Path -Path $prereqSourcePath -ChildPath adkPeAddon

  $prereqFolders=@(
    $sccmPreReqPath,
    $adkPath,
    $adkPePath
  )

  $peAddonPresent=$true
  $peAddonRequired=$false
  [version]$adkVersionNoPE='10.1.17763.1'

  foreach($folder in $prereqFolders){
    $folderExist=Test-Path -Path $folder
    if(!$folderExist){
      switch -Wildcard ($folder) {

        '*adkPeAddon' {

          $peAddonPresent=$false
          continue
        }

        '*adk' {

          Write-Error "Required PreReq folder not found [$folder]" `
            -ErrorAction Stop
        }

        Default {
          $warn=[String]::Format(
            "The folder {0} was not found. {1}.  {2}",
            $folder,
            "This may cause issues during sccm install",
            "Install will continue..."
          )
          Write-Warning $warn
        }
      } # End switch
    } # End folder test
  } # End folder loop


  ### ADK and WinPE Addon Validation
  Write-Verbose "Validating ADK Status"

  $adkMandatoryFeatureList=@(
    'OptionId.DeploymentTools',
    'OptionId.UserStateMigrationTool',
    'OptionId.WindowsPreinstallationEnvironment'
  )
  $requiredAdkFeatureList=New-Object -TypeName Collections.ArrayList

  $adkInstallHash=GetADKInstalledFeatures -ErrorAction SilentlyContinue
  foreach($feature in $adkMandatoryFeatureList){
    if($adkInstallHash.$feature -ne 1){
      Write-Verbose "[$feature] required"
      [void]$requiredAdkFeatureList.Add($feature)
    }
  }
  if($requiredAdkFeatureList){
    Write-Verbose "ADK features missing... Validating ADKSetup files"
    $adk=Get-Item -Path "$adkPath\adksetup.exe" -ErrorAction Stop
    [version]$adkVersion=$adk.VersionInfo.FileVersion

    if($adkVersion -ge $adkVersionNoPE){
      $peAddonRequired=$true

      if(!$peAddonPresent){
        $emsg=[String]::Format(
          "Your version of ADK requires a WinPE Addon. {0}. {1}.",
          "The addon is not present",
          "Unable to install ADK"
        )
        Write-Error $emsg -ErrorAction Stop
      }
      $adkPeSetupFile=Test-Path -Path "$adkPePath\adkwinpesetup.exe"
      if(!$adkPeSetupFile){
        $emsg=[String]::Format(
          "Your version of ADK requires a WinPE Addon. {0}. {1}.",
          "The adkwinpesetup.exe file is not present",
          "Unable to install ADK"
        )
        Write-Error $emsg -ErrorAction Stop
      }
    }
  }else{
    Write-Verbose "All mandatory ADK features present"
  }
  ### END of ADK and WinPE Addon Validation


  ### Windows Feature Validation

  if(!(Get-Module -Name ServerManager -ErrorAction SilentlyContinue)){
    Import-Module -Name ServerManager -ErrorAction Stop
  }
  $features=Get-WindowsFeature -ErrorAction Stop

  $winFeatureList=@(
    'NET-HTTP-Activation',
    'NET-Non-HTTP-Activ',
    'NET-Framework-45-ASPNET',
    'NET-WCF-HTTP-Activation45',
    'NET-WCF-TCP-PortSharing45',
    'BITS',
    'BITS-IIS-Ext',
    'RDC',
    'Web-Server',
    'Web-Common-Http',
    'Web-Default-Doc',
    'Web-Dir-Browsing',
    'Web-Http-Errors',
    'Web-Static-Content',
    'Web-Http-Redirect',
    'Web-Health',
    'Web-Http-Logging',
    'Web-Request-Monitor',
    'Web-Http-Tracing',
    'Web-Security',
    'Web-Filtering',
    'Web-Basic-Auth',
    'Web-CertProvider',
    'Web-IP-Security',
    'Web-Url-Auth',
    'Web-Windows-Auth',
    'Web-App-Dev',
    'Web-Net-Ext',
    'Web-Net-Ext45',
    'Web-ISAPI-Ext',
    'Web-ISAPI-Filter',
    'Web-Includes',
    'Web-Ftp-Server',
    'Web-Ftp-Service',
    'Web-Mgmt-Tools',
    'Web-Mgmt-Console',
    'Web-Mgmt-Compat',
    'Web-Metabase',
    'Web-Lgcy-Mgmt-Console',
    'Web-Lgcy-Scripting',
    'Web-WMI',
    'Web-Scripting-Tools',
    'Web-Mgmt-Service',
    'RSAT-Feature-Tools',
    'RSAT-Bits-Server'
  )

  # NET Framework Core is removed from recent OS's.
  ## Must be installed from Windows install media
  $netfxCoreFeature='NET-Framework-Core'
  $netfx3=$features | Where-Object {$_.Name -eq $netfxCoreFeature}
  if(!($netfx3.Installed)){
    if(!($windowsMediaSourcePath)){
      $mediaWarn=[String]::Format(
        "WindowsFeature: {0}. {1} is required for this installation. {2}.",
        "[$netfxCoreFeature] is not installed",
        "You will need to complete this manually before you can install SCCM."
      )
      Write-Warning $mediaWarn
    }else{
      Write-Verbose "Installing Windows feature: [$netfxCoreFeature]"
      Install-WindowsFeature -Name $netfxCoreFeature `
        -Source $windowsMediaSourcePath `
        -ErrorAction Stop
    }
  }
  ### END Windows Feature Installation

  Write-Verbose "Installing remaining Windows features"
  Install-WindowsFeature -Name $winFeatureList -ErrorAction Stop


  if(!$requiredAdkFeatureList){
    Write-Verbose "ADK Install not required"
    return
  }

  $winPeFeature='OptionId.WindowsPreinstallationEnvironment'

  if(!($requiredAdkFeatureList.Contains($winPeFeature))){
    $peAddonRequired=$false
  }

  if($peAddonRequired){
    $requiredAdkFeatureList.Remove($winPeFeature)
  }

  if($requiredAdkFeatureList){
    $adkFeature=$($requiredAdkFeatureList -join ' ')
    & "$adkPath\adksetup.exe" /ceip off /features $adkFeature /quiet

    WaitForProcessEnd -processName adksetup `
      -msg "Installing ADK"
  }
  if($peAddonRequired){
    & "$adkPePath\adkwinpesetup.exe" /features $winPeFeature /quiet

    WaitForProcessEnd -processName adkwinpesetup `
      -msg "Installing ADK PE Addon"
  }
}Catch{
  Write-Error $_
}
```

#### Example 1

```powershell
SccmPreReqInstall.ps1 -prereqSourcePath . -windowsMediaSourcePath D:\sources\sxs\
```
<p>
  In this example, 
  we are using the current directory for the prereqSourcePath.  
  By using the -windowsMediaSourcePath parameter, 
  we are telling the script to install .NetFramework 3.5 using 
  the Windows installer source media under D:\sources\sxs. 
  This would have required us to mount the 
  Windows installer ISO prior to running the script.
</p>


#### Example 2
```powershell
SccmPreReqInstall.ps1 -prereqSourcePath C:\temp\sccm -Verbose
```

<p>
  In this example, 
  we specify that the ADK prerequisites are in the C:\temp\sccm folder. 
  Since we have not specified -windowsMediaSourcePath, 
  the script will not install .Net Framework 3.5, 
  but it will notify us if it is not installed 
  so we can manually install it later. 
</p>

### What is Left
----

The last bit of setup that is still needed for the SCCM install, 
is the Active Directory portion.
You will still need to create the System Management container in AD, 
and setup the permissions on that container. 
I have made a post / script to do exactly this 
[here]({{ site.baseurl }}/PowerShell-Sccm-AD-PreRequisites/). 
Lastly, you will need to extend the schema. 
<p>
  When you get to running the SCCM installer, 
  just remember to choose the 'prereq' folder 
  (in the prereq source path) when you get to the 
  Prerequisite Downloads step.
</p>
