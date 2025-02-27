# Windows Event Viewer

This document is dedicated to offering various ways to use Event logs to achieve different purposes.

<br>

## How to identify which Windows Firewall rule is responsible for a blocked packet

I've mostly considered this for the [Country IP Blocking category](https://github.com/HotCakeX/Harden-Windows-Security#country-ip-blocking), but you can use it for any purpose.

Before doing this, you need to activate one of the system Audits.

I suggest doing it using GUI because it will have a permanent effect:

![image](https://user-images.githubusercontent.com/118815227/213814954-8ce40aac-bfb0-4973-8677-c77ac232dfb9.png)

<br>

Or you can activate that Audit using this command, but it will only temporarily activate it and it'll be disabled again after you restart Windows.

<br>

```powershell
Auditpol /set /category:"System" /SubCategory:"Filtering Platform Packet Drop" /success:enable /failure:enable
```

<br>

After the Audit is activated, running this PowerShell code will generate an output showing you blocked packets **(if any).**

For example, if you visit a website or access a server that is hosted in one of the countries you blocked, or a connection was made from one of those countries to your device, it will generate an event log that will be visible to you once you run this code.

<br>

```powershell
#Requires -RunAsAdministrator
#Requires -Version 7.3

foreach ($event in Get-WinEvent -FilterHashtable @{LogName = 'Security'; ID = 5152 }) {
    $xml = [xml]$event.toxml()
    $xml.event.eventdata.data | 
    ForEach-Object { $hash = @{ TimeCreated = [datetime] $xml.Event.System.TimeCreated.SystemTime } } { $hash[$_.name] = $_.'#text' } { [pscustomobject]$hash } |
    Where-Object FilterOrigin -NotMatch 'Stealth|Unknown|Query User Default|WSH Default' | ForEach-Object {      
        if ($_.filterorigin -match ($pattern = '{.+?}')) {        
            $_.FilterOrigin = $_.FilterOrigin -replace $pattern, (Get-NetFirewallRule -Name $Matches[0]).DisplayName
        }
        $protocolName = @{ 6 = 'TCP'; 17 = 'UDP' }[[int] $_.Protocol]
        $_.Protocol = if (-not $protocolName) { $_.Protocol } else { $protocolName }
 
        $_.Direction = $_.Direction -eq '%%14592' ? 'Outbound' : 'Inbound'
        $_
    }
}
```

<br>

* [Audit Filtering Platform Packet Drop](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/audit-filtering-platform-packet-drop)
* [Filter origin audit log improvements](https://learn.microsoft.com/en-us/windows/security/threat-protection/windows-firewall/filter-origin-documentation)
* [Audit object access](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/basic-audit-object-access)

<br>

## How to Get Event Logs in Real Time in PowerShell

This code assumes you've already used the [Harden Windows Security Module](https://github.com/HotCakeX/Harden-Windows-Security?tab=readme-ov-file#miscellaneous-configurations) and the event logs custom views exist on your machine.

In this example, any logs generated for Exploit Protection is displayed in real time on PowerShell console. You can modify and improve the displayed output more according to your needs.

```powershell
$LastEventTime = Get-Date

# Comment this entire region if not using xml to specify event source and capture logic
#region XML-Loading

# For when QueryList isn't needed to be extracted
#$FilterXml = Get-Content -Path ".\Exploit Protection Events.xml" -Raw

# Load the XML content from a file or a string
# For Exploit Protection Events
$xml = [xml](Get-Content -Path 'C:\ProgramData\Microsoft\Event Viewer\Views\Hardening Script\Exploit Protection Events.xml')

# Get the QueryList element using XPath
$queryList = $xml.SelectSingleNode('//QueryList')

# Convert the QueryList element to a string
$queryListString = $queryList.OuterXml
#endregion XML-Loading

while ($true) {
    $Events = Get-WinEvent -FilterXml $queryListString -Oldest | Sort-Object -Property TimeCreated -Descending
          
    <#
    For When you don't use xml to specify the event source

    $Events = Get-WinEvent -FilterHashtable @{
        'LogName' = 'Microsoft-Windows-CodeIntegrity/Operational'
        'ID'      = 3077
    } | Sort-Object -Property TimeCreated -Descending
#>

    if ($Events) {
        foreach ($Event in $Events) {
            if ($Event.TimeCreated -gt $LastEventTime) {
                
                Write-Host "`n##################################################" -ForegroundColor Yellow

                $Time = $Event.TimeCreated
                Write-Host "Found new event at time $Time"
                $LastEventTime = $Time

                Write-Host "Message: $($Event.Message)`n" -ForegroundColor Cyan

                # Convert the event to XML
                $Xml = [xml]$Event.toxml()

                # Loop over the data elements in the XML
                $Xml.event.eventdata.data | ForEach-Object -Begin {
                    # Create an empty hash table
                    $DataHash = @{}
                } -Process {
                    # Add a new entry to the hash table with the name and text value of the current data element
                    $DataHash[$_.name] = $_.'#text'
                } -End {
                    # Convert the hash table to a custom object and output it
                    [pscustomobject]$DataHash
                }
                Write-Host '##################################################' -ForegroundColor Yellow
            }
        }
    }
    Start-Sleep -Milliseconds 500
}
```

<br>

If you don't want the real time mode and just want to get the logs one time, you can use the following code

```powershell
# Load the XML content from a file or a string
$xml = [xml](Get-Content -Path 'C:\ProgramData\Microsoft\Event Viewer\Views\Hardening Script\Exploit Protection Events.xml')

# Get the QueryList element using XPath
$queryList = $xml.SelectSingleNode("//QueryList")

# Convert the QueryList element to a string
$queryListString = $queryList.OuterXml

$Events = Get-WinEvent -FilterXml $queryListString -Oldest
$Events | Format-Table -AutoSize
```

<br>
