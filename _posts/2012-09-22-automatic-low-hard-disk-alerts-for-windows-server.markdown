---
layout: post
title: Automated Disk Space Alerts For Windows Server
date: '2012-09-22 15:47:11'
---

<h3>PowerShell Style</h3> <p>I recently needed to automate disk alerts for each of our servers, to keep it simple I wrote a little PowerShell script to check the drive space of each server I’m interested in and send an email if any are below a given threshold. I then configured a scheduled task on one of the servers to run this script every hour. It seems to work quite well so far, touch wood.</p> <p>The script looks like this….</p><pre class="brush: csharp; gutter: false; toolbar: false;">$minGbThreshold = 25;
$computers = "localhost", "server1", "server2";
$smtpAddress = "smtp.myserver.com";
$toAddress = "gavin@gavin.com";
$fromAddress = "alerter@gavin.com";
foreach($computer in $computers)
{    
    $disks = Get-WmiObject -ComputerName $computer -Class Win32_LogicalDisk -Filter "DriveType = 3";
    $computer = $computer.toupper();
    $deviceID = $disk.DeviceID;
    foreach($disk in $disks)
    {
        $freeSpaceGB = [Math]::Round([float]$disk.FreeSpace / 1073741824, 2);
        if($freeSpaceGB -lt $minGbThreshold)
        {
            $smtp = New-Object Net.Mail.SmtpClient($smtpAddress)
            $msg = New-Object Net.Mail.MailMessage
            $msg.To.Add($toAddress)
            $msg.From = $fromAddress
            $msg.Subject = “Diskspace below threshold ” + $computer + "\" + $disk.DeviceId
            $msg.Body = $computer + "\" + $disk.DeviceId + " " + $freeSpaceGB + "GB Remaining";
            $smtp.Send($msg)
        }
    }
}
</pre>
<p>As you can see the top 5 lines are the configuration variables that define the parameters for the script to run within.</p>
<p>To make the script run as a scheduled task without having PowerShell flash up on the server I used the hidden window style argument in the program to run textbox of the Windows Task Scheduler. It looks like this……</p>
<p><em>PowerShell.exe -WindowStyle "Hidden" Script.ps1</em></p>