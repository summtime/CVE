# : Cellinx NVT Webserver Root Privilege Escalation


I. VULNERABILITY
-------------------------
Root privilege escalation in Cellinx Systems' NVT Webserver UR ver. 5.0.0.014b.test, Build at 2019-09-05 allows remote user to run command with root privilege via improper access controls.


II. CVE REFERENCE
-------------------------


III. VENDOR
-------------------------
Cellinx Systems, Ltd. (https://sites.google.com/view/cellinx-systems-eng?)


IV. TIMELINE
-------------------------
2020/01/22 - Vulnerability discovered


V. DESCRIPTION
-------------------------
This vulnerability is caused by the lack of access control in the NVT Webserver which can manage their products through web page.
NVT Web enables only an administrator to access 'Setting' button which is written in Javascript. The attacker can change HTTP requests and responses because authorization check is in only on the client side. 
'GetFileContent.cgi' reads the contents of the file. Since the web server process is running with root privileges, it can read sensitive information such as /etc/passwd.
The attacker can also create a root account on the server without password by using 'SetFileContent.cgi' like below.

![image](https://user-images.githubusercontent.com/4977656/75419446-2f6fea80-5979-11ea-8ce6-974f79f4ec9a.png)


VI. SOLUTION
-------------------------
- Run web server process as an account without root privilege
- Use of a server-side access control


VII. REFERENCES
-------------------------
[CWE-264 Permissions, Privileges, and Access Controls](https://cwe.mitre.org/data/definitions/264.html)
