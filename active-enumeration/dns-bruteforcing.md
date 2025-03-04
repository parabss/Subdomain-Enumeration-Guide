# DNS Bruteforcing

## What is DNS bruteforcing?

It's a technique where the person takes a long list of common subdomain names and append their target to them and based on the response determines whether they are valid or not. This is similar to the dictionary attack.

Below we can what happens in bruteforcing:

* **admin**            ----&gt;       **admin**.target.com
* **internal.dev**  ----&gt;       **internal.dev**.target.com
* **secret**            ----&gt;       **secret**.target.com
* **backup01**      ----&gt;       **backup01**.target.com

Now that we have a list of probable domain names we need to check whether any of these predicted subdomains exist or not. For that, we need to do a mass DNS resolution. After this process, if any of these subdomains is found valid, it's a win-win situation for us.

### Why do we perform subdomain bruteforcing?

At times passive DNS data doesn't give all the hosts/subdomains associated with our target. Also, there would some newer subdomains that still wouldn't have been crawled by the internet crawlers. In such a case subdomain bruteforcing proves beneficial.

Earlier DNS zone transfer vulnerabilities were the key to get the whole DNS infrastructure of a particular organization. But lately, the DNS servers have been secured and zone transfers are found very rarely.

### Problems faced during subdomain bruteforcing

####  1\) Wildcard DNS records

A wildcard DNS record is a record that matches requests for non-existent domain names. Wildcards are denoted by specifying a **`*`** of the left part of a domain name such as **\*.target.com.** That means even if a subdomain doesn't exist it will return a valid response. See the example below:-

**doesntexists.target.com**    ----&gt;   **valid** 

**Strange right?** So in short, if a domain is a wildcard domain we will get all valid responses\(false positives\) while bruteforcing and wouldn't be able to differentiate which are valid and which aren't. To avoid this various wildcard filtering techniques are used by subdomain bruteforcing tools.

**2\) Open Public resolvers**

While bruteforcing we tend to use a long wordlist of common subdomain names to get those hidden domains, hence the domains to be resolved will also be large. Such large resolutions cannot be performed by your system's DNS resolver, hence we depend on freely available public resolvers. Also, using public resolvers eliminates the changes of DNS rate limits.

We can get the list of open public DNS resolvers from here [https://public-dns.info/nameservers.txt](https://public-dns.info/nameservers.txt)

{% hint style="info" %}
📖 Read ****[**this** ](https://app.gitbook.com/@sidxparab/s/subdomain-enumeration-guide/introduction/prequisites#2-100-accurate-public-dns-resolvers)article on why how to create public resolvers and they are important
{% endhint %}

**3\) Bandwidth**

While performing subdomain bruteforcing [massdns](https://github.com/blechschmidt/massdns) is used as a base tool for DNS querying at very high concurrent rates. For this, the underlying system should also possess a higher bandwidth. 

## How to perform subdomain bruteforcing:

### Which tool to choose?

There are currently 2 tools that do the work of DNS bruteforcing and resolution that are [**shuffledns**](https://github.com/projectdiscovery/shuffledns) & [**puredns**](https://github.com/d3mondev/puredns). But there are just some reasons why I prefer puredns over shuffledns.

Table of execution time and false positives.Using an 11 million wordlist on **ibm.com**

| **Tool** | **Shuffledns** | **Puredns** ✅  |
| :--- | :--- | :--- |
| Execution Time | 20m 30s | 9m 32s |
| Output Results | 2900 | 1025 |
| False Positives | 1930 | 0 |

\[ Above tests were performed in separate VPS\[4cpu/8gb\] and false positives were verified using [dnsx ](https://github.com/projectdiscovery/dnsx)with Google DNS server as a trusted resolver-03/06/2021 \]



## [Puredns](https://github.com/d3mondev/puredns)

* **Author:** [d3mondev](https://github.com/d3mondev)
* **Language**: Go
* **Features**: DNS Bruteforcing & Resolution

Puredns outperforms the work of DNS bruteforcing & resolving millions of domains at once.

### Steps Puredns performs:

**1\) Sanitize the input wordlist**

The input wordlist is first sanitized to include only valid characters\(`[a-z0-9.-]`\) and sets all words to lowercase.

**2\) Mass resolve using the public resolvers**

To perform mass resolution of millions of domains at a high speed  [Massdns](https://github.com/blechschmidt/massdns) is used. Massdns using the public DNS resolvers provided; checks whether the probable subdomains generated by us are valid or not by querying those public resolvers. This is generally performed at an unlimited rate and generates a huge amount of traffic.

**3\) Wildcard detection**

As discussed earlier wildcards should be properly detected to eliminate false positives. Puredns then uses its wildcard detection algorithm to detect and extract all the wildcard subdomain roots from the previous massdns output file.

**4\) Validating results with trusted resolvers**

Now the issue with using public DNS resolvers is that they can be [DNS cache poisoned](https://www.cloudflare.com/en-in/learning/dns/dns-cache-poisoning/). This issue arrives because DNSSEC\(a secure way of querying authoritative name servers\) is not widely adopted. To avoid DNS poisoning we verify our final results once again with trusted resolvers like Goggle DNS resolvers \(`8.8.8.8` , `8.8.4.4`\). We can trust the results from trusted resolvers because they use DNSSEC.To avoid over abuse of trusted public DNS resolvers puredns has set a query limit of 500/qps\(queries per second\).

### Installing Puredns:

Since this tool is written in Go, your Go environment should be configured properly.

```bash
GO111MODULE=on go get github.com/d3mondev/puredns/v2
```

### Running Puredns:

Before we start using puredns for bruteforcing we need to generate our public DNS resolvers. For this, we will use a tool called [dnsvalidator](https://github.com/vortexau/dnsvalidator). Check [my previous page](https://app.gitbook.com/@sidxparab/s/subdomain-enumeration-guide/introduction/prequisites#2-100-accurate-public-dns-resolvers) to know more about public DNS resolvers and why they are important.

```bash
git clone https://github.com/vortexau/dnsvalidator.git
cd dnsvalidator/
python3 setup.py install
```

**Generating list of open public resolvers**

 It's very important to note that even if one of your public resolver is failing/not working you have a greater chance of missing an important subdomain. Hence, it's always advised that you generate a fresh public DNS resolvers list before execution.

```bash
dnsvalidator -tL https://public-dns.info/nameservers.txt -threads 100 -o resolvers.txt
```

![](../.gitbook/assets/dnsvalidator1.png)

Now that we have generated our public DNS resolver we are good to move ahead and perform subdomain bruteforcing using puredns.

```bash
puredns bruteforce wordlist.txt example.com -r resolvers.txt -w output.txt
```

**Flags:**

* **bruteforce** - use the bruteforcing mode
* **r** - Specify your public resolvers
* **w** - Output filename

![](../.gitbook/assets/purednsb.png)

{% hint style="success" %}
While performing DNS queries sometimes we receive **SERVFAIL** error. Puredns by default retries on SERVFAIL while most tools don't.
{% endhint %}

### Which wordlist 📄 to use?

The whole idea DNS bruteforcing is of no use if you don't use a great wordlist. Selection of the wordlist is the most important aspect of bruteforcing. Let's look at what best wordlist:-  
  
**1\) Assetnote** [**best-dns-wordlist.txt**](https://wordlists-cdn.assetnote.io/data/manual/best-dns-wordlist.txt) \(**9 Million**\) ⭐  
[Assetnote](https://wordlists.assetnote.io/) wordlists are the best. No doubt this is the best subdomain bruteforcing wordlist. But highly recommended that you run this in your VPS. Running on a home system will take hours also the results wouldn't be accurate. This wordlist will definitely give you those hidden subdomains.

**2\) Jhaddix** [**all.txt**](https://gist.github.com/jhaddix/f64c97d0863a78454e44c2f7119c2a6a) \(**2 Million**\)  
Created by the great [Jhaddix](https://twitter.com/Jhaddix). Was last updated 2 years ago but still works well.

**3\) Smaller** [**wordlist**](https://gist.github.com/six2dez/a307a04a222fab5a57466c51e1569acf/raw) \(**102k** \)  
Created by [six2dez](https://github.com/six2dez) is suitable to be run on home systems.  


### Issues faced and how to overcome them: 👊 

#### 1\) Crashes on low specs\( 1cpu/1gb vps\)

Usually, if you provide a very large wordlist\(50M\) and your target contains significant wildcards then sometimes puredns crashes out due to less memory while filtering wildcards. To overcome this issue you can use **`--wildcard-batch 1000000`** flag. By default, puredns puts all the domains in a single batch to save on the number of DNS queries and execution time. Using this flag takes in a batch of only 1million subdomains at a time for wildcard filtering and after completion of the task takes in the next batch for wildcard filtering.

**2\) Puredns kills my home router** 

Massdns is the one to be blamed for. Massdns tries to perform DNS resolution using public resolvers at an unlimited rate. This generates large traffic and makes your home router unable to use for that specific period of time. To overcome this you can use the **`-l`** flag. This flag throttles the massdns threads to your specified amount. It's advisable that you set the value anywhere between `2000-10000`

\`\`





