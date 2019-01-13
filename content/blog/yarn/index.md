---
title: Minor yarn upgrade glitch
date: '2019-01-13'
---

Noticed this line while using yarn to install redux-logger
```
warning Your current version of Yarn is out of date. The latest version is "1.13.0", while you're on "1.12.3"
info To upgrade, run the following command: 
$ sudo apt-get update && sudo apt-get install yarn 
```
Trying the command provided in the log, resulted in this output
```bash
Err:10 https://dl.yarnpkg.com/debian stable InRelease
The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 4F77679369475BAA
```
A simple google search directed me to this github [issue](https://github.com/yarnpkg/yarn/issues/4453).
Ran this command mentioned both in the issue and the yarn installation page:
```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
```
Re-running the command suggested by yarn above (update and install), upgraded yarn successfully!
Rather than just running this command, it might be helpful to understand what exactly this command does
* [curl](https://curl.haxx.se/docs/manpage.html) is a command line tool and library for transferring data with URLs.
* The s flag (or --silent) runs curl in silent mode. 
* S (or --show-error) flag with s, makes curl show an error message when running it in silent mode
* The pipe (|) command takes the output from the left side, and passes it as an input to the right side. So the 
input from 
```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg
```
will be used as an output to the command
```
sudo apt-key add - 
``` 
dash (-) is used to signal the end of options. More info [here](https://askubuntu.com/a/703434) 
* apt-key manages the key ring of gpg files used by secure apt to authenticate packages. We downloaded the public key 
for the yarn package. If we run sudo apt-key list, after adding the key, we should see an entry similar to this:
```
/etc/apt/trusted.gpg
....
pub   4096R/86E50310 2016-10-05
uid                  Yarn Packaging <yarn@dan.cx>
sub   4096R/D50AF136 2016-10-05
sub   4096R/88B63E1E 2019-01-02 [expires: 2020-02-02]
sub   4096R/69475BAA 2019-01-11 [expires: 2020-02-02]
```
 * A very detailed explanation is found in the secureApt section of the ubuntu community help [page]
 (https://help.ubuntu.com/community/SecureApt). A gist 
 of this for our purpose is this: 
 ```
 A private key is used to sign the package, and we use the public key to verify if the package 
 was signed by the private key. gpg, which stands for GNU Privacy Guard, is used to sign the files and check
 for signatures.
 ```