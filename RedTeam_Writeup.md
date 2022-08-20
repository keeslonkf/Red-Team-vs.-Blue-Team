###Penetration Test Engagement in a Virtualized Environment

There are several steps involved with penetrating a network: Planning and Reconnaissance, Scanning, Exploitation, Post Exploitation, and Reporting. Once the network has been exploited there are a number of ways an attacker can abuse a system including stealing sensitive data, modifying data on the network, compromising the availability of data for a ransom and leaving a backdoor for perpetual access.

![RedTeam_NetworkDiagram]

This document contains the following details:
- Description of the Topology
- Vulnerability Assessment
- Exploitation
- Post-Exploitation

###Description of Topology

In this lab, there were several machines on the network to be tested. A Windows VM which contained Hyper-V Manager served as the gateway to other nested VMs on the same network. A machine called Capstone served as a vulnerable Web server. Another VM on the network was an ELK server which is used to capture log data from the Capstone Machine. This data is then presented using the Kibana platform. This platform is useful for parsing logs, creating visual representations of data, and creating alerts. Lastly, for this engagement the actual attacking machine - Kali - was on the network itself as well. The Capstone VM is the proposed target for this engagement. The goal of infiltrating the Capstone machine was to set up a reverse shell in which the Kali attack machine could remotely access the system. The sole target that was infiltrated was the Capstone VM.

The configuration details of each machine may be found below.

| Hostname | Function       | IP Address               | Operating System |
|----------|----------------|--------------------------|------------------|
| Gateway  | Gateway        | 192.168.1.1              | Windows 10 Pro   |
| Kali     | Attack Machine | 192.168.1.90             | Linux            |
| ELK      | ELK Log Server | 192.168.1.100            | Linux            |
| CAPSTONE | Target Machine | 192.168.1.105            | Linux            |

###Vulnerability Assessment

The assessment uncovered the following critical vulnerabiliies in the target:

- Public Facing Sensitive Data: Files alluding to the existence of a secret folder within the web server
  - Impact: Gives attackers incentive to probe for more sensitive data
- Brute Force Attack and Hash cracking: Attackers can use programs to guess login credentials or decrypt password hashes to gain credentials
  - Impact: Discovered credentials can be used to potentially access sensitive data on the web server
- Web Misconfiguration: Security Controls are not in place to restrict who can upload files to the web server
  - Impact: Attackers can upload and plant malware to later be used on the server
- Local File Inclusion (LFI): Attackers can execute the planted malware file to potentially gain a reverse shell on the web server
  - Impact: Discovered credentials can be used to potentially access sensitive data on the web server       

###Exploitation

Vulnerabilities were identified in the target by first site mapping the web server using a simple Firefox Browser and a Kali Linux tool called dirb. Dirb revealed an important URL on the server:"/webdav". ![dirb_scan] Through a public-facing sensitive data vulnerability, it was discovered that a secret directory hidden within the web server existed and was only accessible by employees. ![Secret_Folder] This prompted the idea of brute forcing employee credentials using the employees listed in the site’s “Meet Our Team” page. The intention was to find credentials to access this secret folder. This brute force attack was successfully executed using the Kali Linux tool “hydra” to obtain the credentials of employee Ashton and gain access to the secret folder. ![hydra] ![secretFolderLogin]

The next exploit was discovered in a file within the secret folder which contained the hash of the CEO’s password and instructions on how to access another hidden directory within the server called “webdav”. ![secret_folderFile] Using a tool called Crack Station online, this hash was cracked and privileges were escalated further. ![crackstation] With this high-level access, a php script was uploaded to the web server to initiate the first step in establishing a reverse shell between the Capstone Server and the Kali attacking machine. ![php_script] The ability to upload a .php script is a web misconfiguration vulnerability that was exploited. 

Finally, an LFI vulnerability ensured the success of exploiting the reverse shell vulnerability by allowing the script to be executed on the server. This was completed using Metasploit in conjunction with the php script. ![metasploit] Once inside, a shell was established and the command line was used to search for a flag. To find the flags, a shell was first established using the command “shell” in the Meterpreter session. Once inside, basic linux commands like “cd” and “find” were utilized to search for files containing the word “flag” in the title.![flag_txt]

###Post Exploitation

There are several ways to reconnect to the target. First, if no backdoor has been established, an attacker would have to exploit the target again using the same steps as before - this is why it is imperative to document every step of the engagement in the event you must recreate a scenario. Creating backdoors is another way to reconnect to a target after being disconnected. This can be accomplished through many methods and it differs based on the OS of the target machine. 

In the case of the Meterpreter session, meterpreter has a script called “persistence.rb” to maintain an open backdoor even if the target system is rebooted. In the meterpreter command line prompt run the command “ run persistence -h”. It is important to note that this method requires no authentication so anyone with access to the port used for the exploit can access the machine. This is not ideal for the purposes of penetration testing unless it is done on a network that is not public-facing. 

Another technique to create a backdoor once inside of the Capstone VM is to use SSH keys; note this service was shown to be running on Port 22 in the nmap service scan so it can be used, however this would not work where this service is not enabled. Simply add your public SSH key into the “~/.ssh/authorized_keys” file to allow for remotely connecting via SSH. 
