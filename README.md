# FiPloit-writeup ---> https://hackerdna.com/labs/fiploit
HackerDNA FiPloit Lab Writeup

Introduction

In this lab, the goal was to obtain both a user and root flag from a vulnerable web application.
The challenge demonstrated a realistic attack chain involving:
*Service discovery
*Directory enumeration
*Information disclosure
*Insecure file upload
*Remote Code Execution (RCE)
*Privilege escalation via sudo misconfiguration
This write-up walks through the full process step by step.

Initial Access – Identifying the Service

The target application was not accessible via the default HTTP port (80).
Access was only possible by specifying port 8080:
http://TARGET_IP:8080
This indicates the web service was running on a non-standard port, a common setup in development environments. In a real engagement, this would typically be discovered using:
nmap -sC -sV -p- TARGET_IP
Once port 8080 was identified as open, the web application became accessible.

Directory & File Enumeration

Next step: directory brute-forcing. Using Gobuster with a file-focused wordlist:

gobuster dir -u http://TARGET_IP:8080/ \
-w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt \
-x txt,php,log,bak \
--exclude-length <length>

This revealed several resorces including:

/notes.txt
/upload_log_temp4.php

The discovery of notes.txt was critical. 

Information Disclosure – Developer Notes
Accessing:

http://TARGET_IP:8080/notes.txt

Revealed internal developer notes such as:
*A temporary upload endpoint not removed before production
*Database credentials (marked to be changed)
*Debug and configuration reminders
This represents Sensitive Information Disclosure. Most importantly, it referenced:
/upload_log_temp4.php
This indicated a potential attack surface via file upload.

Exploiting the File Upload Functionality

The upload page allowed only .txt files. Attempting to upload:
shell.php   ---> was blocked

However, the validation only checked whether .txt appeared in the filename.
It did not enforce it as the sole extension.
Uploading:
shell.txt.php

successfully bypassed the filter.
Why this worked ?
The application checked for .txt
Apache interpreted the last extension
The final extension was .php
The file was executed by the PHP interpreter
This is a classic Improper File Extension Validation vulnerability.

Achieving Remote Code Execution (RCE):
The uploaded file contained a simple PHP command execution payload (modified here for documentation safety):
<?p_h_p system($_GET['cmd']); ?>
Accessing:
http://TARGET_IP:8080/uploads/shell.txt.php?cmd=whoami
Returned the server user.
At this moment, Remote Code Execution was achieved.
The attack flow was:
HTTP request sent with cmd parameter
Apache passed file to PHP interpreter
PHP executed system command
Output returned in HTTP response
This provided full command execution on the server.

User Enumeration & User Flag

Once RCE was obtained, system enumeration began:
id
pwd
ls /home
The user context was confirmed, and navigating to the user’s home directory revealed: flag-user.txt
Reading the file via the web shell retrieved the user flag.

Privilege Escalation

Next step: check sudo permissions.
sudo -l
Revealed:
User may run /usr/bin/php without password
This is a critical misconfiguration.
Allowing a user to execute PHP via sudo is equivalent to granting root shell access.
Using PHP CLI:
sudo php -r 'system("id");'
Confirmed execution as a root. The root flag was then retrieved from the root directory.

Full Exploitation Chain

The complete attack path:
1.Service discovery on port 8080
2.Directory enumeration
3.Discovery of developer notes
4.Identification of temporary upload endpoint
5.File upload bypass via double extension
6.Web shell upload
7.Remote Code Execution
8.Sudo misconfiguration abuse
9.Root privilege escalation
10.This demonstrates how multiple small misconfigurations can combine into full system compromise.

Conclusion

This lab was an excellent demonstration of:
Realistic web exploitation
Logical attack chaining
Proper post-exploitation enumeration
Privilege escalation through misconfiguration
The most important takeaway is not the individual commands, but the reasoning process behind each step.
Small oversights in development can lead to complete system compromise.
