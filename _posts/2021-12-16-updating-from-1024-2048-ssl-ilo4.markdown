---
title: "Updating from 1024-bit to 2048-bit SSL Keys on HPE iLO 4"
layout: post
date: 2021-12-16
image: /assets/images/2021-12-16-updating-from-1024-2048-ssl-ilo4/broken-padlock.png
headerImage: true
tag:
- openstack
- ironic
- hpe
- proliant
- ssl
- ilo
category: blog
blog: true
author: jamesdenton
description: "Updating from 1024-bit to 2048-bit SSL Keys on HPE iLO 4."
---

A recent attempt to move away from IPMI to the native HPE iLO 4 driver in my OpenStack Ironic lab showed just how wrong I was to believe it would be a seamless change. What I found was that while ironic-conductor could communicate with iLO, apparently, it didn't like what it saw:

```
[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: EE certificate key too weak
```
<!--more-->
No big deal, right? There should be something obvious in the iLO page to alert me to this weakness, and a check and a click and I'd be back in business.

Wrong.

Even moving from the `ECDHE-RSA-DES-CBC3-SHA` cipher to `ECDHE-RSA-AES256-GCM-SHA384` by enabling AES in iLO wasn't enough to get things moving. I had to dig *deeper*.

## Unnamed Internet Hero

A little bit of Googling and I came across something [interesting](https://itsjustbytes.wordpress.com/2020/04/22/hp-ilo-4-certificate-upgrade-from-1024-bit-to-2048-bit/):

> There is an update for ILO 4 that incorporates a new 2048 bit certificate
    
My new friend **roadglide03** gave me the hint I needed, along with an upgrade script and some RPMs. A cursory glance at the Perl didn't reveal anything suspicious, so off I went.

## Getting Started

To follow the process to the letter, one would download HPE's [Lights-Out Online Configuration Utility for Linux](https://support.hpe.com/hpesc/public/swd/detail?swItemId=MTX_640d4499d8c64ee79f546d439f) [here](https://downloads.hpe.com/pub/softlib2/software1/pubsw-linux/p215998034/v182899/hponcfg-5.6.0-0.x86_64.rpm). This link provides an RPM that may need to be extracted with `rpm2cpio`:

```
# rpm2cpio hponcfg-5.6.0-0.x86_64.rpm | cpio -id
```

Or, for the Ubuntu folks, the deb works just as well:

```
# curl https://downloads.linux.hpe.com/SDR/hpPublicKey2048.pub | apt-key add -
# curl https://downloads.linux.hpe.com/SDR/hpPublicKey2048_key1.pub | apt-key add -
# curl https://downloads.linux.hpe.com/SDR/hpePublicKey2048_key1.pub | apt-key add -

# add-apt-repository 'deb http://downloads.linux.hpe.com/SDR/repo/mcp focal/current non-free'
# add-apt-repository 'deb http://downloads.linux.hpe.com/SDR/repo/mcp focal/12.20 non-free'
# apt-get update

# apt-get install hponcfg 
```

The thing to know about `hponcfg` is that it allows one to modify the **local** iLO only. Fine if you have an OS on your machine and have a handful to manage. Not fine if you have a fleet and/or no operating system (more on that later).

## Using replaceSSLcert.pl

The `replaceSSLcert.pl` works with the following switches:

- --check
- --update

Using `--check`, you ought to end up with a message like this:

```
# perl replaceSSLcert.pl --check

Here's the output:

Pre Check/Update Info Gathering
    Gathering info from the local iLO
        ILO IP: 172.19.0.27
        ILO DNS NAME: lab-infra01-ilo
        ILO DOMAIN NAME: shands.local
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
WARNING: ILO DNS Domain name does not match local domain
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
WARNING: ILO IP (172.19.0.27) resolves to

    which does not match configured ILO DNS Name
        lab-infra01-ilo
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Checking certificate for lab-infra01-ilo
CERTIFICATE UPDATE NEEDED
lab-infra01-ilo(172.19.0.27) certificate is only 1024 bits long
Which is less than the minimum length of 2048 bits.
```

Which is to say, there are a lot of complaints here about the state of iLO on this machine. Whatever, I don't really care, I just want a 2048-bit key:

```
lab-infra01-ilo(172.19.0.27) certificate is only 1024 bits long
Which is less than the minimum length of 2048 bits.
```

The process of updating the key is handled by the script, and it will work through the following:

- Generate CSR
- Create a CA
- Generate a 'signed' key
- Upload PEM to iLO

To help generate a CSR with at least some accurate information, the following blocks in the `replaceSSLcert.pl` should be updated to reflect the proper values:

```
<CSR_STATE VALUE ="TX"/>
<CSR_COUNTRY VALUE ="US"/>
<CSR_LOCALITY VALUE ="San Antonio"/>
<CSR_ORGANIZATION VALUE ="jimmdenton"/>
<CSR_ORGANIZATIONAL_UNIT VALUE ="lab"/>
<CSR_COMMON_NAME VALUE ="lab-infra01-ilo.jimmdenton.com"/>
```

In addition, you should update `/etc/hosts` on the machine running `replaceSSLcert.pl` with an entry for iLo and the short *and* common name:

```
172.19.0.27     lab-infra01-ilo lab-infra01-ilo.jimmdenton.com
```

To verify things work as expected, you can run `openssl` to verify the key size before and after the change:

```
# echo | openssl s_client -connect 172.19.0.27:443 2>/dev/null | openssl x509 -text -noout | grep "Public-Key"
                RSA Public-Key: (1024 bit)
```
                
Now you can run the script with `--update`:

```
# perl replaceSSLcert.pl --update
Pre Check/Update Info Gathering
	Gathering info from the local iLO
        ILO IP: 172.19.0.27
        ILO DNS NAME: lab-infra01-ilo
        ILO DOMAIN NAME: shands.local
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
WARNING: ILO DNS Domain name does not match local domain
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
	ILO DNS Name matches ILO IP DNS Lookup (lab-infra01-ilo)
Checking certificate for lab-infra01-ilo
lab-infra01-ilo(172.19.0.27) certificate is only 1024 bits long
Which is less than the minimum length of 2048 bits.
Issuing openssl genrsa command
Issuing openssl req command
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
WARNING: ILO DNS Domain:
	arcanebyte.com
         DOES NOT MATCH LOCAL DOMAIN:
	openstack.local
THIS NEEDS TO BE FIXED ON THE LOCAL ILO
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
About to update the local iLO certificate with:
	FQDN: lab-infra01-ilo

ARE YOU SURE YOU WANT TO CONTINUE?
PLEASE ANSWER 'YES' or 'NO':
```

Answering `YES` will allow the script to proceed with the aforementioned steps. The process takes about 30 seconds, give or take, and will result in iLO being reset - so you you can expect to lose access if you're already logged in.

To verify, run `openssl` again:

```
# echo | openssl s_client -connect 172.19.0.27:443 2>/dev/null | openssl x509 -text -noout | grep "Public-Key"
                RSA Public-Key: (2048 bit)
```

Sweet! This should rid me of the pesky `certificate key too weak` error. Now, how to scale this operation.

## Remote iLO (or using locfg.pl)

To scale this out across the lab (about 9 nodes) I wanted to find a way to hit iLO over the network rather than locally. More Googling and a Little Bit of Luck™ let me to the [HPE Lights-Out XML PERL Scripting Sample for Linux](https://support.hpe.com/hpesc/public/swd/detail?swItemId=MTX_ac3efba097b84fe8acfcbdd5d5) which provides (yet another) Perl script for managing iLO remotely: `locfg.pl`.

The download provides a ton of XML files along with the Perl script itself, which we will use to verify it actually works. But first, you may need to install a pre-requisite:

```
# apt install libsocket6-perl
```

Running the script, we can see what's necessary to make it go:

```
# ./locfg.pl

Usage: perl locfg.pl -s server -f inputfile [options]
       perl locfg.pl -s ipV4Address -f inputfile [options]
       perl locfg.pl -s ipV4Address:portNumber -f inputfile [options]
       perl locfg.pl -s ipV6Address -f inputfile [options]
       perl locfg.pl -s [ipV6Address] -f inputfile [options]
       perl locfg.pl -s [ipV6Address]:portNumber -f inputfile [options]
       perl locfg.pl -s DnsName:portnumber -f inputfile [options]
    -l logfile         log file
    -v                 enable verbose mode
    -t                 substitute variables with values specified(ab=xy,c=z)
    -i                 entering username and password interactively
    -u username        username
    -p password        password
    -ilo3|-ilo4|-ilo5  target is iLO 3, iLO 4 or iLO 5

  Note: Use -u and -p with caution as command line options are
        visible on Linux. The '-i' option is for entering the
        username and password interactively.
```

To test, I'll try to get all users using `Get_All_Users.xml`: 

```
<RIBCL VERSION="2.0">
   <LOGIN USER_LOGIN="adminname" PASSWORD="password">
      <USER_INFO MODE="read">
         <GET_ALL_USERS/>
      </USER_INFO>
   </LOGIN>
</RIBCL>
```

To execute looks like this:

```
# perl locfg.pl -s 172.19.0.24 -u root -p <password> -ilo4 -f Get_All_Users.xml
<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
<GET_ALL_USERS>
    <USER_LOGIN VALUE="Administrator"/>
    <USER_LOGIN VALUE="maas"/>
    <USER_LOGIN VALUE="root"/>
</GET_ALL_USERS>
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

...Script Succeeded...
```

Looks good! Now comes the fun part of constructing everything that `replaceSSLcert.pl` did for us.

### Generating Things

The server I hope to attack first is **texas04**, a baremetal node used with Ironic that does not have an operating system installed. 

The first step is to generate an RSA key that will be used for the CA used to sign the new 2048-bit certificate/key generated for iLO on **texas04**:

From **lab-infra01**:

```
# mkdir /tmp/texas04/
# /usr/bin/openssl genrsa -out /tmp/texas04/myCA.key 2048 2>/dev/null
```

Then, generate a CA:

```
# /usr/bin/openssl req -x509 -new -nodes -key /tmp/texas04/myCA.key -sha256 -days 3650 -out /tmp/texas04/myCA.pem -subj "/C=US/ST=TX/L=San Antonio/O=jimmdenton/OU=lab/CN=US ORG" 2>/dev/null
```

Next, we want iLO on **texas04** to generate a CSR using attributes we've defined here (which you should change). The `locfg.pl` script will be used to trigger iLO to do the needful:

```
# cat <<EOF >> /tmp/texas04/csr.xml
<RIBCL VERSION="2.0">
<LOGIN USER_LOGIN = "USERID" PASSWORD = "PASSW0RD">
<RIB_INFO MODE="write">
<!-- Default -->
<!-- <CERTIFICATE_SIGNING_REQUEST/> -->
<!-- Custom CSR -->
<CERTIFICATE_SIGNING_REQUEST>
<!-- Change the following to match your needs -->
<CSR_STATE VALUE ="TX"/>
<CSR_COUNTRY VALUE ="US"/>
<CSR_LOCALITY VALUE ="San Antonio"/>
<CSR_ORGANIZATION VALUE ="jimmdenton"/>
<CSR_ORGANIZATIONAL_UNIT VALUE ="lab"/>
<CSR_COMMON_NAME VALUE ="texas04-ilo.jimmdenton.com"/>
</CERTIFICATE_SIGNING_REQUEST>
</RIB_INFO>
</LOGIN>
</RIBCL>
EOF
```

Running this:

```
# perl locfg.pl -s 172.19.0.24 -u root -p <password> -ilo4 -f /tmp/texas04/csr.xml
```

Results in this:

```
The iLO subsystem is currently generating a Certificate Signing Request(CSR), run script after 10 minutes or more to receive the CSR.
```

I waited maybe 2 minutes before proceeding, but you can check the status with this command:

```
# perl locfg.pl -s 172.19.0.24 -u root -p <password> -ilo4 -f /tmp/texas04/csr.xml -l /tmp/texas04/csr.out
```

When the CSR is ready, it will be reflected in `/tmp/texas04/csr.out`:

```
# cat /tmp/texas04/csr.out
<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
<CERTIFICATE_SIGNING_REQUEST>
-----BEGIN CERTIFICATE REQUEST-----
MIIC7TCCAdUCAQAwdDEfMB0GA1UEAwwWdGV4YXMwNC5hcmNhbmVieXRlLmNvbTEM
MAoGA1UECwwDTGFiMRMwEQYDVQQKDApBcmNhbmVCeXRlMRQwEgYDVQQHDAtTYW4g
QW50b25pbzELMAkGA1UECAwCVFgxCzAJBgNVBAYTAlVTMIIBIjANBgkqhkiG9w0B
AQEFAAOCAQ8AMIIBCgKCAQEAuQUaPlY4LdIEecqYWEy6WHk4p/J5WyNyJ9o01l/R
dtrfquYsBgNWMZqVRJt8FgCbbLqTUBH+C+aB1E34BPxcKFBvIG2bYQuFf+aokPNc
RuXR8/0pOodQtMJQYrpCZwJMnU6CrDQ5aIl0NCiSOxU6HSxnS/Bkly2PR64JjgWq
bv5794MQQUXtP4bxhOodlJaIVagCenklSIm8xN+/dfjkZdtjo/yVSF79a/DokbNb
iiX+zLCQO11OjCFTJMBC2aub4F2Q9D6fqaAKgp8mdykGLM2GJBvKYEMzqv5/RrcE
qc2I8Uc6CjreDYApYDgsNrEuNG1XhnaeE8P1jBeqhExKvwIDAQABoDQwMgYJKoZI
hvcNAQkOMSUwIzAhBgNVHREEGjAYghZ0ZXhhczA0LmFyY2FuZWJ5dGUuY29tMA0G
CSqGSIb3DQEBCwUAA4IBAQBd7Zxy8Suo48csSDkoLxLnG3Z6zeqNvjAlnENVUfHg
IkKGctpPbzVSvZUJj+uaXGDsjJeg/Qwptab2PU/E2j/QPqt/9bNtl7eEdlqXaGHJ
qJoSL+wi4mO2/wczdax7QLvSvCtJ+HvDKIXwq1ra7cuThlosWjQhUzhKJCrK6PAH
xNcOhJxIGld41To+kH98YPJoWDq4GsD9Fl48OIpjr0ItDo3htGahKOMsinqgDfjc
GFsxEKDrVkDf8iD+7gHgs+VHtslkG5Bz+pIFeza9M4MmKPGaitlUR6K+j7ZiAs/L
N3EP5Ti6I6iUd8SA79i1wFhUzoyqDTk7UURatLu07XyX
-----END CERTIFICATE REQUEST-----
</CERTIFICATE_SIGNING_REQUEST>
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

...Script Succeeded...
```

The important bits lie between the brackets:

```
-----BEGIN CERTIFICATE REQUEST-----
MIIC7TCCAdUCAQAwdDEfMB0GA1UEAwwWdGV4YXMwNC5hcmNhbmVieXRlLmNvbTEM
MAoGA1UECwwDTGFiMRMwEQYDVQQKDApBcmNhbmVCeXRlMRQwEgYDVQQHDAtTYW4g
QW50b25pbzELMAkGA1UECAwCVFgxCzAJBgNVBAYTAlVTMIIBIjANBgkqhkiG9w0B
AQEFAAOCAQ8AMIIBCgKCAQEAuQUaPlY4LdIEecqYWEy6WHk4p/J5WyNyJ9o01l/R
dtrfquYsBgNWMZqVRJt8FgCbbLqTUBH+C+aB1E34BPxcKFBvIG2bYQuFf+aokPNc
RuXR8/0pOodQtMJQYrpCZwJMnU6CrDQ5aIl0NCiSOxU6HSxnS/Bkly2PR64JjgWq
bv5794MQQUXtP4bxhOodlJaIVagCenklSIm8xN+/dfjkZdtjo/yVSF79a/DokbNb
iiX+zLCQO11OjCFTJMBC2aub4F2Q9D6fqaAKgp8mdykGLM2GJBvKYEMzqv5/RrcE
qc2I8Uc6CjreDYApYDgsNrEuNG1XhnaeE8P1jBeqhExKvwIDAQABoDQwMgYJKoZI
hvcNAQkOMSUwIzAhBgNVHREEGjAYghZ0ZXhhczA0LmFyY2FuZWJ5dGUuY29tMA0G
CSqGSIb3DQEBCwUAA4IBAQBd7Zxy8Suo48csSDkoLxLnG3Z6zeqNvjAlnENVUfHg
IkKGctpPbzVSvZUJj+uaXGDsjJeg/Qwptab2PU/E2j/QPqt/9bNtl7eEdlqXaGHJ
qJoSL+wi4mO2/wczdax7QLvSvCtJ+HvDKIXwq1ra7cuThlosWjQhUzhKJCrK6PAH
xNcOhJxIGld41To+kH98YPJoWDq4GsD9Fl48OIpjr0ItDo3htGahKOMsinqgDfjc
GFsxEKDrVkDf8iD+7gHgs+VHtslkG5Bz+pIFeza9M4MmKPGaitlUR6K+j7ZiAs/L
N3EP5Ti6I6iUd8SA79i1wFhUzoyqDTk7UURatLu07XyX
-----END CERTIFICATE REQUEST-----
```

That CSR should get saved in it's own file:

```
cat <<EOF >> /tmp/texas04/real-csr.out
-----BEGIN CERTIFICATE REQUEST-----
MIIC7TCCAdUCAQAwdDEfMB0GA1UEAwwWdGV4YXMwNC5hcmNhbmVieXRlLmNvbTEM
MAoGA1UECwwDTGFiMRMwEQYDVQQKDApBcmNhbmVCeXRlMRQwEgYDVQQHDAtTYW4g
QW50b25pbzELMAkGA1UECAwCVFgxCzAJBgNVBAYTAlVTMIIBIjANBgkqhkiG9w0B
AQEFAAOCAQ8AMIIBCgKCAQEAuQUaPlY4LdIEecqYWEy6WHk4p/J5WyNyJ9o01l/R
dtrfquYsBgNWMZqVRJt8FgCbbLqTUBH+C+aB1E34BPxcKFBvIG2bYQuFf+aokPNc
RuXR8/0pOodQtMJQYrpCZwJMnU6CrDQ5aIl0NCiSOxU6HSxnS/Bkly2PR64JjgWq
bv5794MQQUXtP4bxhOodlJaIVagCenklSIm8xN+/dfjkZdtjo/yVSF79a/DokbNb
iiX+zLCQO11OjCFTJMBC2aub4F2Q9D6fqaAKgp8mdykGLM2GJBvKYEMzqv5/RrcE
qc2I8Uc6CjreDYApYDgsNrEuNG1XhnaeE8P1jBeqhExKvwIDAQABoDQwMgYJKoZI
hvcNAQkOMSUwIzAhBgNVHREEGjAYghZ0ZXhhczA0LmFyY2FuZWJ5dGUuY29tMA0G
CSqGSIb3DQEBCwUAA4IBAQBd7Zxy8Suo48csSDkoLxLnG3Z6zeqNvjAlnENVUfHg
IkKGctpPbzVSvZUJj+uaXGDsjJeg/Qwptab2PU/E2j/QPqt/9bNtl7eEdlqXaGHJ
qJoSL+wi4mO2/wczdax7QLvSvCtJ+HvDKIXwq1ra7cuThlosWjQhUzhKJCrK6PAH
xNcOhJxIGld41To+kH98YPJoWDq4GsD9Fl48OIpjr0ItDo3htGahKOMsinqgDfjc
GFsxEKDrVkDf8iD+7gHgs+VHtslkG5Bz+pIFeza9M4MmKPGaitlUR6K+j7ZiAs/L
N3EP5Ti6I6iUd8SA79i1wFhUzoyqDTk7UURatLu07XyX
-----END CERTIFICATE REQUEST-----
EOF
```

Now, we generate the PEM:

```
# /usr/bin/openssl x509 -req -in /tmp/texas04/real-csr.out -CA /tmp/texas04/myCA.pem -CAkey /tmp/texas04/myCA.key -CAcreateserial -out /tmp/texas04/CRT.pem -days 3650 -sha256"
```

Output:

```
Signature ok
subject=CN = texas04.arcanebyte.com, OU = lab, O = jimmdenton, L = San Antonio, ST = TX, C = US
Getting CA Private Key
```

Verify:

```
# cat /tmp/texas04/CRT.pem
-----BEGIN CERTIFICATE-----
MIIDXzCCAkcCFFJKSKj/1ixZEyQEWKTbFs9DOQ66MA0GCSqGSIb3DQEBCwUAMGQx
CzAJBgNVBAYTAlVTMQswCQYDVQQIDAJUWDEUMBIGA1UEBwwLU2FuIEFudG9uaW8x
EzARBgNVBAoMCkFyY2FuZUJ5dGUxDDAKBgNVBAsMA0xhYjEPMA0GA1UEAwwGVVMg
T1JHMB4XDTIxMTIxNjA0MzQwNloXDTMxMTIxNDA0MzQwNlowdDEfMB0GA1UEAwwW
dGV4YXMwNC5hcmNhbmVieXRlLmNvbTEMMAoGA1UECwwDTGFiMRMwEQYDVQQKDApB
cmNhbmVCeXRlMRQwEgYDVQQHDAtTYW4gQW50b25pbzELMAkGA1UECAwCVFgxCzAJ
BgNVBAYTAlVTMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuQUaPlY4
LdIEecqYWEy6WHk4p/J5WyNyJ9o01l/RdtrfquYsBgNWMZqVRJt8FgCbbLqTUBH+
C+aB1E34BPxcKFBvIG2bYQuFf+aokPNcRuXR8/0pOodQtMJQYrpCZwJMnU6CrDQ5
aIl0NCiSOxU6HSxnS/Bkly2PR64JjgWqbv5794MQQUXtP4bxhOodlJaIVagCenkl
SIm8xN+/dfjkZdtjo/yVSF79a/DokbNbiiX+zLCQO11OjCFTJMBC2aub4F2Q9D6f
qaAKgp8mdykGLM2GJBvKYEMzqv5/RrcEqc2I8Uc6CjreDYApYDgsNrEuNG1Xhnae
E8P1jBeqhExKvwIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQCXPFYDn69ceirgt5TR
i6iBgIsVDEcuFmSj72krf+dTrlt1JYtUVFyRYdLw3MaWy186JF3emq2lvPEyU6SA
fnOSM2lBrxF0LDZ9QpkOb+PWZE1JRthzE5Xxg6q5oUPbR/XJuFLljkg9hz60v5Xd
pGNXcV/Hh4S6EBELfQ94ju73rvuRK149VYSp9TMpzja5GEyKH9xHDgfG+GK/siDB
JlyLlmSwr3PeNZwtwB+rZmkjzxzBvsp9CQSuNiLN6B12OeD946MuvJcQ6hhXkImY
WwURDSE8sII4XYeLT9+4D1gPWbBDAkx5kUCgVqE4jtn232MCGd3Md+4ek23S+Stz
wW0O
-----END CERTIFICATE-----
```

Now, we can tuck the certificate into an XML file for upload:

```
cat <<EOF >> /tmp/texas04/2048cert.xml
<RIBCL VERSION="2.0">
<LOGIN USER_LOGIN = "USERID" PASSWORD = "PASSW0RD">
<RIB_INFO MODE = "write">
<IMPORT_CERTIFICATE>
-----BEGIN CERTIFICATE-----
MIIDXzCCAkcCFFJKSKj/1ixZEyQEWKTbFs9DOQ66MA0GCSqGSIb3DQEBCwUAMGQx
CzAJBgNVBAYTAlVTMQswCQYDVQQIDAJUWDEUMBIGA1UEBwwLU2FuIEFudG9uaW8x
EzARBgNVBAoMCkFyY2FuZUJ5dGUxDDAKBgNVBAsMA0xhYjEPMA0GA1UEAwwGVVMg
T1JHMB4XDTIxMTIxNjA0MzQwNloXDTMxMTIxNDA0MzQwNlowdDEfMB0GA1UEAwwW
dGV4YXMwNC5hcmNhbmVieXRlLmNvbTEMMAoGA1UECwwDTGFiMRMwEQYDVQQKDApB
cmNhbmVCeXRlMRQwEgYDVQQHDAtTYW4gQW50b25pbzELMAkGA1UECAwCVFgxCzAJ
BgNVBAYTAlVTMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuQUaPlY4
LdIEecqYWEy6WHk4p/J5WyNyJ9o01l/RdtrfquYsBgNWMZqVRJt8FgCbbLqTUBH+
C+aB1E34BPxcKFBvIG2bYQuFf+aokPNcRuXR8/0pOodQtMJQYrpCZwJMnU6CrDQ5
aIl0NCiSOxU6HSxnS/Bkly2PR64JjgWqbv5794MQQUXtP4bxhOodlJaIVagCenkl
SIm8xN+/dfjkZdtjo/yVSF79a/DokbNbiiX+zLCQO11OjCFTJMBC2aub4F2Q9D6f
qaAKgp8mdykGLM2GJBvKYEMzqv5/RrcEqc2I8Uc6CjreDYApYDgsNrEuNG1Xhnae
E8P1jBeqhExKvwIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQCXPFYDn69ceirgt5TR
i6iBgIsVDEcuFmSj72krf+dTrlt1JYtUVFyRYdLw3MaWy186JF3emq2lvPEyU6SA
fnOSM2lBrxF0LDZ9QpkOb+PWZE1JRthzE5Xxg6q5oUPbR/XJuFLljkg9hz60v5Xd
pGNXcV/Hh4S6EBELfQ94ju73rvuRK149VYSp9TMpzja5GEyKH9xHDgfG+GK/siDB
JlyLlmSwr3PeNZwtwB+rZmkjzxzBvsp9CQSuNiLN6B12OeD946MuvJcQ6hhXkImY
WwURDSE8sII4XYeLT9+4D1gPWbBDAkx5kUCgVqE4jtn232MCGd3Md+4ek23S+Stz
wW0O
-----END CERTIFICATE-----
</IMPORT_CERTIFICATE>
<!-- The iLO will be reset after the certificate has been imported. -->
<MOD_GLOBAL_SETTINGS>
<ENFORCE_AES VALUE="Y"/>
<IPMI_DCMI_OVER_LAN_ENABLED VALUE="N"/>
</MOD_GLOBAL_SETTINGS>
<RESET_RIB/>
</RIB_INFO>
</LOGIN>
</RIBCL>
EOF
```

Before the upload commences, check the current state:

```
# echo | openssl s_client -connect 172.19.0.24:443 2>/dev/null | openssl x509 -text -noout | grep "Public-Key"
                RSA Public-Key: (1024 bit)
```

Then, run `locfg.pl` with the new XML:

```
# perl locfg.pl -s 172.19.0.24 -u root -p <password> -ilo4 -f /tmp/texas04/2048cert.xml
```

Output:

```
root@lab-infra01:~/hp# perl locfg.pl -s 172.19.0.24 -u root -p <password> -ilo4 -f /tmp/texas04/2048cert.xml
<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
<INFORM>Integrated Lights-Out will reset at the end of the script.</INFORM>
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='AES is already Enabled.'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
<INFORM>Integrated Lights-Out will reset at the end of the script.</INFORM>
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

<?xml version="1.0"?>
<RIBCL VERSION="2.23">
<RESPONSE
    STATUS="0x0000"
    MESSAGE='No error'
     />
</RIBCL>

...Script Succeeded...
```

After iLO resets, another look shows the change is successful:

```
# echo | openssl s_client -connect 172.19.0.24:443 2>/dev/null | openssl x509 -text -noout | grep "Public-Key"
                RSA Public-Key: (2048 bit)
```
   
## Summary

Like everything I do, it is usually preceded by a day's worth of unexpected, yet related, tasks to get things to a state where I can actually get done what I wanted to get done. And then, that is followed up by new errors that allow the process to repeat ad-nauseum. I'm sure you can all relate.

I don't think it would take much to update the `replaceSSLcert.pl` script to use `locfg.pl` to interface with remote iLO, since it handles the bulk of the generation of SSL-related bits and would really save time. Better yet, Ansible could be used to make it a pretty quick process. I've made both available Perl scripts on my [GitHub](https://github.com/busterswt/hp_tooling) for anyone that needs them. 

---
If you have some thoughts or comments on this article, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.
