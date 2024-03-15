---
title: Cisco Expressway DNS and Ports
author: Will
type: posts
date: 2017-01-20T21:18:00+00:00
slug: cisco-expressway-dns-and-ports
aliases: /2017/01/cisco-expressway-dns-and-ports/
categories:
  - Cisco Unified Communications
  - voice
summary: Today I was working with a client who was facing some MRA issues for their 8841 handsets and Jabber. I've found that typically the issues are around certificates, but this client's MRA worked sporadically. I pondered this for a few moments and asked the Network Admin to check the ports on the firewall.. as it turned out, he didn't have access as this was tightly controlled by security.
---

Today I was working with a client who was facing some MRA issues for their 8841 handsets and Jabber. I’ve found that typically the issues are around certificates, but this client’s MRA worked sporadically. I pondered this for a few moments and asked the Network Admin to check the ports on the firewall.. as it turned out, he didn’t have access as this was tightly controlled by security.

As much UC consultants know, network security handled by non-network saavy people, usually turns out to be a complete pain in the ass. No change here. We'll bulletize the conversation for ease of reading, it starts with me.

  * nmap the IP from the outside
  * what’s nmap?
  * ok telnet to each port on the ip from the outside
  * I can’t find telnet on my machine and I’m not an admin
  * OK I’ll write a script

I took a moment to decide what this script needed to test for, inevitably, scope creep set in. First, I needed to verify the proper SRV records were created in the external DNS. I thought, ‘Well, if I’m going to check for the proper records, let’s check and ensure that the wrong records don’t exist.’ I added in a couple common mistakes I see, such as \_cisco-uds and \_cuplogin existing in external DNS (these should ONLY be internal).  
Next I thought, OK we need to check ports. I made a list of ports that are required to be open from the Internet through the firewall to the expressway-e. Again, I thought, ‘Well, what about ports that SHOULDN’T be open?’ So I added the functionality in to check for ports that should NOT be open to the outside.  
I’ll add that I’m still working on this script, but the base functionality is achieved and I wanted to discuss it before I forget.

Below is the current iteration of the code.

```powershell
$srvlist=@("_h323ls._udp", "_sip._udp", "_sip._tcp", "_h323cs._tcp", "_sips._tcp", "_collab-edge._tls", "_xmpp-server._tcp", "_cuplogin._tcp", "_cisco-uds._tcp")
$portlist=@(21, 22, 23, 25, 80, 110, 442, 443, 1719, 1720, 5060, 5061, 5222, 8443)
$domain=read-host "What is the domain you are checking SRV records for? eg. prosysis.com"
$allowed=@(1719, 1720, 5060, 5061, 5222, 8443)
$wellknown=@(21,22,23,25,80,110,443)
$srvdom=@()
$portsopen=@()
$portsclosed=@()

write-host ""
write-host ""
write-host ""
write-host ""
write-host ""
Write-Host "Checking required SRV records for $($domain)" 
Write-Host "--------------------------------------------" 
foreach ($srv in $srvlist){
	try{
		$DNS=Resolve-DnsName -Name "$($srv).$($domain)" -Type srv -DnsOnly -ErrorAction Stop
		$location = $DNS.NameTarget
		write-host "$($srv) was found on $location" -foregroundcolor "green"  
		if(!($srvdom.Contains($location))){
			$srvdom += $location
		}
	} Catch{
		write-host "$($srv) was NOT found on $($domain)" -foregroundcolor "red"  
	}
}
write-host ""
Write-Host "Checking required Port State for $($domain) "  
Write-Host "--------------------------------------------"  
$portsobhm=@()
$portsocha=@()
$portscbhm=@()
$portsccha=@()
$warnportbhm=@()
$warnportcha=@()
foreach ($server in $srvdom | select -uniq){
	$i=0
	foreach ($port in $portlist){
		$i++
		Write-Progress -Activity "Scanning Ports" -status "Scanning Port $($port) on $($server)" -percentComplete ($i / $portlist.count*100)
		Try{
			$tcp=New-Object System.Net.Sockets.TcpClient
			$tcp.Connect($server, $port)
			if($server -like "pro-bhm*"){
				if($allowed -contains $port){$portsobhm+=$port}
				if($wellknown -contains $port){if(!$warn){$warn=1};$warnportbhm+=$port}
				if(!$server1){$server1=$server}
			}else{
				if($allowed -contains $port){$portsocha+=$port}
				if($wellknown -contains $port){if(!$warn){$warn=1};$warnportcha+=$port}
				if(!$server2){$server2=$server}
			}
		}Catch{
			if($server -like "pro-bhm*"){
				if($allowed -contains $port){$portscbhm+=$port}
				if(!$server1){$server1=$server}
			}else{
				if($allowed -contains $port){$portsccha+=$port}
				if(!$server2){$server2=$server}
			}			
		}
	}
}
Write-Host "Successful TCP Connections:"  
if($portsobhm){Write-Host "TCP Port(s) $($portsobhm) on $($server1)" -foregroundcolor "green" } 
if($portsocha){Write-Host "TCP Port(s) $($portsocha) on $($server2)" -foregroundcolor "green" } 
Write-Host ""
Write-Host "Failed TCP Connections:"  
if(($portscbhm) -or ($portsccha)){Write-Host "Please make sure the following ports are opened on the firewall." }
if($portscbhm){Write-Host "TCP Port(s) $($portscbhm) on $($server1)" -foregroundcolor "red" }
if($portsccha){Write-Host "TCP Port(s) $($portsccha) on $($server2)" -foregroundcolor "red" }
Write-Host ""
if($warn){Write-Host "The following well-known ports were found to be open:"}
if($warn){Write-Host "For security purposes, it is best practice to have these ports blocked on the firewall."}
if($warnportbhm.count -gt 0){
	Write-Host -nonewline "TCP Port(s) " -backgroundcolor "yellow" -foregroundcolor "red"
	for($j=1; $j -le $warnportbhm.count; $j++){
		Write-Host $warnportbhm&#91;$j] -backgroundcolor "yellow" -foregroundcolor "red"
	}
}
if($warnportcha.count -gt 0){
	Write-Host -nonewline "TCP Port(s) " -backgroundcolor "yellow" -foregroundcolor "red"
	for($j=1; $j -le $warnportcha.count; $j++){
		Write-Host $warnportcha&#91;$j] -backgroundcolor "yellow" -foregroundcolor "red"
	}
}
```

The part that’s not working is the end where I’m outputting the ports that shouldn’t be open. It’s simply not outputting the data and honestly, I haven’t really looked at it other than to run it and provide the results to the Network Admin. My plans are to get that piece working and also break out proper vs improper SRV records. Currently, it just says if they exist without mentioning if the record shouldn’t exist.

You might be saying right now, well, Cisco offers a tool that does this exactly, and your output looks just like it. Well, obviously I wrote my script to look like the output of the expressway test tool Cisco has. In this particular case, the network admin in question didn’t have access to log into Cisco and use the tool. It’s a pretty simple thing to write and now he can do checks whenever he wants. Amusingly, I started writing this in bash because… well it would’ve run faster, but of course, most network admins have no access to a Linux shell, let alone any experience with it. (which is honestly surprising considering how much shit runs on Linux these days)