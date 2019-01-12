---
title: Yarn 101
date: '2019-01-12'
---

## Upgrade yarn
Noticed this line when installing redux-logger using yarn
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
A simple google search led me to this github issue page: https://github.com/yarnpkg/yarn/issues/4453.
Tried a command outlined in the issue, and got it to work:
```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
```
Rerunning the command suggested by yarn, upgraded yarn successfully!
Rather than just running this command, it might be helpful to understand what exactly this command does
* curl is a command line tool and library for transferring data with URLs (from the official curl library)

* using the s flag with curl, runs it in silent mode. using S (capital S) with s, makes curl show an error message
when running it in silent mode

* apt-key manages the list of keys used by apt to authenticate packages. We basically add a key to apt to authenticate the yarn
package.

