---
layout: post
title: EAPHammer Version 0.7.0 - Certificate Handling
categories:
- wireless
- eaphammer
---

The latest version of EAPHammer greatly expands its ability to generate, import, and manage private keys and x509 certificates. This post describes these new features in detail and provides the necessary background information to understand why these new features were needed.

- [https://github.com/s0lst1c3/eaphammer/releases/tag/v0.7.0-beta](https://github.com/s0lst1c3/eaphammer/releases/tag/v0.7.0-beta)

# Background

EAPHammer was initially created as a wrapper to hostapd-wpe, which uses vanilla hostapd’s bootstrap script to create self-signed server certificates. The bootstrap script, in turn, relies on a series of configuration files that must be edited by hand. Managing these configuration files can be painful to deal with when operating under the kinds of time constraints that are typical in security consulting. The earliest releases of EAPHammer addressed these usability issues by providing a feature known as Cert Wizard, which acted as a streamlined interface to hostapd's bootstrap script.

Cert Wizard consisted of a series of prompts that asked the user to input attributes such as organization, organizational unit, and company email address. These attributes were then written to bootstrap's configuration files as the certificate's Common Name (CN) and Subject Alternative Name (SAN). Cert Wizard would then run the bootstrap script to generate a self-signed server certificate. This process is shown in Figure 1.

<img src="http://s0lst1c3.github.io/images/eaphammer-v0.7.0/cert-wizard-blog-old-cw.gif" alt="drawing" width="600"/>

*Figure 1*

EAPHammer's Cert Wizard offered multiple advantages that were somewhat unique when it was released (although it was only a matter of time until someone else built a tool with similar features). Not only did it make it possible to stand up a rogue access point within seconds, rather than minutes, but it did so in a way that was effective against the vast majority of vulnerable devices (although EAPHamer inherited this second advantage from hostapd-wpe, which means credit goes to Brad Antoniewicz). However, the original version of Cert Wizard also suffered from several glaring limitations:

### One size fits all approach

Cert Wizard was only capable of creating self-signed certificates with a very narrow set of attributes. Additionally, users were afforded no control over things like key length. Choice of cryptographic algorithms was determined by a hardcoded variable buried deep within EAPHammer's (rather frightening) codebase.

### No support for certificates created outside of EAPHammer

There are a lot of situations in which it makes sense to use a server certificate that was obtained elsewhere:

* You’d like to use a believable certificate chain cloned directly from a target using a tool such as Apostille [\[1\]](http://solstice.kennethsun.net/wireless/eaphammer/2019/04/18/eaphammer-v0-7-0-certificate-handling/#references).

* You'd like to use compromised certificates that you’ve managed to obtain from the target organization. 

* You’d like to use a valid certificate signed by an external CA such as [Let’s Encrypt](https://letsencrypt.org/). Some badly designed (or poorly configured) supplicants actually accept these certificates as valid  [\[2\]](http://solstice.kennethsun.net/wireless/eaphammer/2019/04/18/eaphammer-v0-7-0-certificate-handling/#references). More commonly, platforms such as OSX will present these certificates to users in such a way that makes them seem trustworthy, even though trusting a certificate signed by an external CA makes zero sense when it's issued by a RADIUS server (see *Figure 2*).

<img src="http://s0lst1c3.github.io/images/eaphammer-v0.7.0/eaphammer-v070-trusted-cert-osx.png" alt="drawing" width="600"/>

*Figure 2*

### Lack of control over Diffie-Hellman (DH) parameters
OpenSSL requires pre-computed DH parameters to support cipher suites with forward secrecy [\[3\]](http://solstice.kennethsun.net/wireless/eaphammer/2019/04/18/eaphammer-v0-7-0-certificate-handling/#references)
. Hostapd stores its DH Parameters in a DH file, which is generated or regenerated each time the bootstrap script is run. Since the original Cert Wizard was a wrapper for the bootstrap script, earlier versions of EAPHammer handled DH parameters in this manner as well.

There are a few problems with this approach. For one thing, the default configs used by the bootstrap script use a DH length of 1024 bits, which falls before the modern recommended length of 2048 bits. In most cases, this isn’t a big deal, but can lead to OpenSSL errors in some situations [\[4\]](http://solstice.kennethsun.net/wireless/eaphammer/2019/04/18/eaphammer-v0-7-0-certificate-handling/#references). Additionally, this approach doesn’t provide the user with a means of manually specifying the DH length, or the ability to regenerate the DH parameters should the need arise.

# Version 0.7.0 Addresses These Issues
First and most importantly, version 0.7.0 completely strips away any reliance on hostapd’s bootstrap scripts. Instead, certificates are created and managed natively by EAPHammer using Python’s OpenSSL and pem libraries [\[5\]](http://solstice.kennethsun.net/wireless/eaphammer/2019/04/18/eaphammer-v0-7-0-certificate-handling/#references)[\[6\]](http://solstice.kennethsun.net/wireless/eaphammer/2019/04/18/eaphammer-v0-7-0-certificate-handling/#references). This made it possible to create a much more fleshed-out version of Cert Wizard with a number of new capabilities.

To start with, Cert Wizard’s certificate creation capabilities have been expanded:

* Self-Signed Certificate Creation - Additional granular controls have been provided for advanced users.
* Trusted Certificate Creation - Cert Wizard can now be used to create trusted server certificates using compromised CA certificates and keys.
* CLI Support - Users are no longer limited to interactive prompts: the entire certificate creation process can be completed as a CLI one-liner as shown in *Figure 3*. Note that Cert Wizard’s original behavior has been preserved in Interactive Mode, which is Cert Wizard’s default mode of operation.

<img src="http://s0lst1c3.github.io/images/eaphammer-v0.7.0/cert-wizard-create-new.gif" alt="drawing" width="600"/>

*Figure 3*

Additionally, version 0.7.0 adds support for externally created certificates, which can either be imported permanently into EAPHammer’s static configuration (*Figure 4*) or used on a one-off basis.

<img src="http://s0lst1c3.github.io/images/eaphammer-v0.7.0/eaphammer-cert-handling-import.gif" alt="drawing" width="600"/>

*Figure 4*

Finally, EAPHammer now relies on a 2048 bit DH file by default. Additionally, users can regenerate the DH file as needed, as well as specify the DH file’s length manually as shown in *Figure 5*.

<img src="http://s0lst1c3.github.io/images/eaphammer-v0.7.0/gif-dh-param.gif" alt="drawing" width="600"/>

*Figure 5*


# References

- [\[1\] https://sensepost.com/blog/2017/recreating-certificates-using-apostille/](https://sensepost.com/blog/2017/recreating-certificates-using-apostille/)
- [\[2\] https://arts.units.it/retrieve/handle/11368/2915044/203999/2018-CS-EvilTwinsSecurityDisaster.pdf](https://arts.units.it/retrieve/handle/11368/2915044/203999/2018-CS-EvilTwinsSecurityDisaster.pdf)
- [\[3\] https://wiki.openssl.org/index.php/Diffie-Hellman_parameters](https://wiki.openssl.org/index.php/Diffie-Hellman_parameters)
- [\[4\] https://github.com/s0lst1c3/eaphammer/issues/69](https://github.com/s0lst1c3/eaphammer/issues/69)
- [\[5\] https://pypi.org/project/pyOpenSSL/](https://pypi.org/project/pyOpenSSL/)
- [\[6\] https://pypi.org/project/pem/](https://pypi.org/project/pem/)
