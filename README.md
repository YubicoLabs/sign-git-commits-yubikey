# sign-git-commits-yubikey
Sign git commits with a YubiKey

Note: This approach uses a YubiKey to store your master certify key. There are many ways to ultimately achieve your end goal. Each has their own strengths and weaknesses. Do your own research and pick the appropriate strategy for your specific requirements.

This guide will walk you through how to generate GPG keys that are good for general use, including encryption and code signing. All keys are generated and stored on YubiKeys. One will be your “master key” - protect it as if it were your actual identity. The other will contain sub-keys that you will use for your day to day use.

# Overview of steps
* Install and configure necessary software
* Set up “master” YubiKey for safekeeping
    * Configure PIN/Admin PIN
    * Generate master key
* Set up “developer” YubiKey for every day use
    * Configure PIN/Admin PIN
    * Generate developer keys (making use of the first YubiKey)
* Adding a new GPG key to your GitHub account
* Setup git to sign commits and tags
* Signing commits
* Verifying commits
* Signing tags
* Verifying tags
* Merging and pushing

# Introduction to OpenPGP
OpenPGP is a standard for using cryptographic keys for various uses. The main implementation of this standard is the GNU Privacy Guard (gnupg) software. In OpenPGP an individual has an "OpenPGP key", which is actually a set of public-private key pairs grouped together under a master key. The master key is the main key pair. Other key pairs are known as subkeys, and any sub-key belonging to the "OpenPGP key" will be signed by the master key. In addition to the master key, it is common to have 3 sub-keys with different usage:

**Signature key** - Used for signing git commits, files, e-mails, etc. to prove that they came from you.

**Encryption key** - Used to encrypt/decrypt stuff like files or e-mails so that only you can see them.

**Authentication key** - Used to authenticate things like an SSH session.

The master key itself signs your own sub-keys, as well as other people's public keys, which is used to indicate that you’ve verified the identity of the key holder. Your master key is also used to extend the expiration date of your own OpenPGP key.

The YubiKey can store 3 key pairs in the OpenPGP application. Typical use will have each of the three sub-keys (Signature, Encryption, and Authentication) on a single YubiKey.

# Prerequisites
## Hardware
You will need two YubiKey 5 series. USB A or C does not matter, and should be determined by what ports you have readily available on your computer. Nano form factor is not recommended as you should always unplug your development key when it is not in use.
## Software
What you need depends on your operating system.
### Linux
Install the following packages via your distribution’s package manager:
* GPG
* pcscd
* YubiKey Manager
For example, on apt based distributions like Ubuntu or Debian:
```
$ sudo apt install pcscd gnupg2
$ sudo add-apt-repository ppa:yubico/stable
$ sudo apt update
$ sudo apt install yubikey-manager
```
### Windows
**YubiKey Manager**

Download the latest version directly from https://www.yubico.com/

**GPG4Win**

Install GPG4Win from https://www.gpg4win.org/download.html - the default settings are fine.

### macOS
**YubiKey Manager**
* Download the latest version directly from https://www.yubico.com/

**GnuPG**

The easiest way to install GnuPG on macOS is using Homebrew:

Note: The latest version of GnuPG (3.2.1 at the time of writing) on Homebrew works fine.
```
$ brew install gnupg
```

**Pinentry**

It’s recommended to install the graphical pinentry program for macOS. 
```
$ brew install pinentry-mac
```
Add to your ~/.gnupg/gpg-agent.conf file:
```
pinentry-program /usr/local/bin/pinentry-mac
```
Add to your ~/.gnupg/scdaemon.conf file:

`disable-ccid` (From the man page: Disable the integrated support for CCID compliant readers. This allows falling back to one of the other drivers even if the internal CCID driver can handle the reader.)

# Generate your master key
This key is not intended for everyday use, it will be used to generate the keys you use day to day. The Yubikey storing it is typically kept in a safe place in between key generations.
From your operating system’s native terminal window, run the following set of commands:
 
Note: If you get a ‘command not found’ or similar error when attempting to run ‘gpg’, make sure that the executable is defined in your environment’s PATH variable.

Warning: GPG does not handle multiple smart cards well. Make sure you only have one YubiKey plugged in at a time, and you have no other smart card reader devices visible on your computer.
```
$ gpg --card-edit

Reader ...........: Yubico YubiKey OTP FIDO CCID 0
Application ID ...: D2760001240102010006078005150000
Version ..........: 2.1
Manufacturer .....: Yubico
Serial number ....: 07800515
Name of cardholder: [not set]
Language prefs ...: [not set]
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 0 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```
You will now be in a special gpg/card> prompt. Enter all subsequent commands here.
```
gpg/card> admin
Admin commands are allowed
```

## Setting your PIN, Admin PIN, and Reset Code
You need to change the various default PINs on the YubiKey. Pick something unique and consider using a password manager for storing them.
```
gpg/card> passwd
gpg: OpenPGP card no. D2760001240102010006078005150000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1
<< Enter PIN. (Default is 123456) >>
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
<< Enter admin PIN. (Default is 12345678) >>

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? Q
```
## Generate your master certify key
### Changing to better defaults
We want to make sure we’re using the strongest key types that are available for GPG. For our purposes, we need to use RSA keys for all key types. Set the key size to the maximum supported by the YubiKey (4096 bits).
```
gpg/card> key-attr
Changing card key attribute for: Signature key
Please select what kind of key you want:
   (1) RSA
   (2) ECC
Your selection? 1
What keysize do you want? (2048) 4096
The card will now be re-configured to generate a key of 4096 bits
Changing card key attribute for: Encryption key
Please select what kind of key you want:
   (1) RSA
   (2) ECC
Your selection? 1
What keysize do you want? (2048) 4096
The card will now be re-configured to generate a key of 4096 bits
Changing card key attribute for: Authentication key
Please select what kind of key you want:
   (1) RSA
   (2) ECC
Your selection? 1
What keysize do you want? (2048) 4096
The card will now be re-configured to generate a key of 4096 bits
```
### Key creation
Now we should create our master certify key. Be sure to add your name and email address as this is used to identify your key.

Warning: When you type 'O' for (O)kay, the YubiKey will begin to generate the keys. This command can take 4-5 minutes. GPG will not give any feedback until this process has completed. Please be patient!
```
gpg/card> generate
Make off-card backup of encryption key? (Y/n) n

Please note that the factory settings of the PINs are
   PIN = '123456'     Admin PIN = '12345678'
You should change them using the command --change-pin

Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: You
Name must be at least 5 characters long
Real name: You McEngineer
Email address: you@example.com
Comment:
You selected this USER-ID:
    "You McEngineer <you@example.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
gpg: key 109F2428DD97A597 marked as ultimately trusted
gpg: revocation certificate stored as 'C:/Users/you/AppData/Roaming/gnupg/openpgp-revocs.d\2F28DCB202028A5A2FE5A45D109F2428DD97A597.rev'
public and secret key created and signed.
```

### Verification of steps so far
Quit and restart the program. If everything is completed successfully, you should see that three new keys have been added.
```
gpg/card> list

Reader ...........: Yubico YubiKey OTP CCID 0
Application ID ...: D2760001240103030006000152110000
Version ..........: 3.3
Manufacturer .....: Yubico
Serial number ....: 00015211
Name of cardholder: [not set]
Language prefs ...: [not set]
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: not forced
Key attributes ...: rsa4096 rsa4096 rsa4096
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 4
KDF setting ......: on
Signature key ....: 2F28 DCB2 0202 8A5A 2FE5  A45D 109F 2428 DD97 A597
      created ....: 2019-12-03 01:21:36
Encryption key....: 4E14 0FFF B296 D2D5 6CD0  A654 C821 9CCE 0DAB FC09
      created ....: 2019-12-03 01:21:36
Authentication key: 529E FBFD BF0C 5908 79A5  4FAB 23DC 6210 FD32 B9BF
      created ....: 2019-12-03 01:21:36
General key info..:
pub  rsa4096/109F2428DD97A597 2019-12-03 You McEngineer <you@example.com>
sec>  rsa4096/109F2428DD97A597  created: 2019-12-03  expires: never
                                card-no: 0006 00015211
ssb>  rsa4096/23DC6210FD32B9BF  created: 2019-12-03  expires: never
                                card-no: 0006 00015211
ssb>  rsa4096/C8219CCE0DABFC09  created: 2019-12-03  expires: never
                                card-no: 0006 00015211
```

### Trim the excess from your master key
Our master key should really only ever be used for certification of our sub-keys and for signing public keys that we trust. We can, and should, get rid of the encryption and authentication keys as they should not be used in day to day operations.
```
$ gpg --edit-key you@example.com

gpg (GnuPG) 2.2.16; Copyright (C) 2019 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   3  signed:   1  trust: 0-, 0q, 0n, 0m, 0f, 3u
gpg: depth: 1  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 1f, 0u
gpg: next trustdb check due at 2020-03-05
Secret key is available.

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
ssb  rsa4096/23DC6210FD32B9BF
     created: 2019-12-03  expires: never       usage: A
     card-no: 0006 00015211
ssb  rsa4096/C8219CCE0DABFC09
     created: 2019-12-03  expires: never       usage: E
     card-no: 0006 00015211
[ultimate] (1). You McEngineer <you@example.com>

gpg> key 1

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
ssb* rsa4096/23DC6210FD32B9BF
     created: 2019-12-03  expires: never       usage: A
     card-no: 0006 00015211
ssb  rsa4096/C8219CCE0DABFC09
     created: 2019-12-03  expires: never       usage: E
     card-no: 0006 00015211
[ultimate] (1). You McEngineer <you@example.com>

gpg> delkey
Do you really want to delete this key? (y/N) y

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
ssb  rsa4096/C8219CCE0DABFC09
     created: 2019-12-03  expires: never       usage: E
     card-no: 0006 00015211
[ultimate] (1). You McEngineer <you@example.com>

gpg> key 1

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
ssb* rsa4096/C8219CCE0DABFC09
     created: 2019-12-03  expires: never       usage: E
     card-no: 0006 00015211
[ultimate] (1). You McEngineer <you@example.com>

gpg> delkey
Do you really want to delete this key? (y/N) y

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
[ultimate] (1). You McEngineer <you@example.com>

gpg> quit
Save changes? (y/N) y
```

# Generate your development sub keys
Now we’ll work on creating the GPG keys that you’ll use on a day-to-day basis. This will utilize the second YubiKey you have - but don’t put away your master key yet! We’re still going to need it to generate your sub keys.

Insert the YubiKey that you want to use as your development key.

Before running the following steps, you should edit the GPG card like you did with the first yubikey and change the PINs (Section Setting your PIN, Admin PIN, and Reset Code) and change key-len defaults (Section Changing to better defaults). Note that these PINs should be different from the PINs used for your master key.

Once that’s out of the way, let’s get started!

## Adding a signature key
```
$ gpg --edit-key you@example.com
gpg (GnuPG) 2.2.16; Copyright (C) 2019 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
[ultimate] (1). You McEngineer <you@example.com>

gpg> addcardkey
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]

Please select the type of key to generate:
   (1) Signature key
   (2) Encryption key
   (3) Authentication key
Your selection? 1

<After select type, asks for subkey yubikey pin>

Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 2y
Key expires at 12/01/21 17:33:39 Pacific Standard Time
Is this correct? (y/N) y
Really create? (y/N) y

<After Really Create asks for subkey yubikey admin pin>

<After generation, it will ask for the master yubikey, remove the subkey yubikey and insert the master>
<Will ask for master yubikey pin to "certify" the new key>
<For signing keys only, gpg will ask you to reinsert the subkey yubikey -- there doesn't appear to be any reason for this>

<When this process is complete, you should see...>

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
ssb  rsa4096/C0EF4AD6881062AC
     created: 2019-12-03  expires: 2021-12-02  usage: S
     card-no: 0006 00015212
[ultimate] (1). You McEngineer <you@example.com>

gpg> save
```
## Adding an encryption key
```
$ gpg --edit-key you@example.com
gpg (GnuPG) 2.2.16; Copyright (C) 2019 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
ssb  rsa4096/C0EF4AD6881062AC
     created: 2019-12-03  expires: 2021-12-02  usage: S
     card-no: 0006 00015212
[ultimate] (1). You McEngineer <you@example.com>

gpg> addcardkey
Signature key ....: 8637 5137 5E90 916D FDC6  B4B4 C0EF 4AD6 8810 62AC
Encryption key....: [none]
Authentication key: [none]

Please select the type of key to generate:
   (1) Signature key
   (2) Encryption key
   (3) Authentication key
Your selection? 2
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 2y
Key expires at 12/01/21 17:36:32 Pacific Standard Time
Is this correct? (y/N) y
Really create? (y/N) y

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
ssb  rsa4096/C0EF4AD6881062AC
     created: 2019-12-03  expires: 2021-12-02  usage: S
     card-no: 0006 00015212
ssb  rsa4096/E0187BC2C0A16274
     created: 2019-12-03  expires: 2021-12-02  usage: E
     card-no: 0006 00015212
[ultimate] (1). You McEngineer <you@example.com>

gpg> save
```

## Adding an authentication key
```
$ gpg --edit-key you@example.com
gpg (GnuPG) 2.2.16; Copyright (C) 2019 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
ssb  rsa4096/C0EF4AD6881062AC
     created: 2019-12-03  expires: 2021-12-02  usage: S
     card-no: 0006 00015212
ssb  rsa4096/E0187BC2C0A16274
     created: 2019-12-03  expires: 2021-12-02  usage: E
     card-no: 0006 00015212
[ultimate] (1). You McEngineer <you@example.com>

gpg> addcardkey
Signature key ....: 8637 5137 5E90 916D FDC6  B4B4 C0EF 4AD6 8810 62AC
Encryption key....: 6DD4 6A9C EFCE EE0D FB79  B5BF E018 7BC2 C0A1 6274
Authentication key: [none]

Please select the type of key to generate:
   (1) Signature key
   (2) Encryption key
   (3) Authentication key
Your selection? 3
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 2y
Key expires at 12/01/21 17:39:46 Pacific Standard Time
Is this correct? (y/N) y
Really create? (y/N) y

sec  rsa4096/109F2428DD97A597
     created: 2019-12-03  expires: never       usage: SC
     card-no: 0006 00015211
     trust: ultimate      validity: ultimate
ssb  rsa4096/C0EF4AD6881062AC
     created: 2019-12-03  expires: 2021-12-02  usage: S
     card-no: 0006 00015212
ssb  rsa4096/E0187BC2C0A16274
     created: 2019-12-03  expires: 2021-12-02  usage: E
     card-no: 0006 00015212
ssb  rsa4096/435E88A6E7960EE5
     created: 2019-12-03  expires: 2021-12-02  usage: A
     card-no: 0006 00015212
[ultimate] (1). You McEngineer <you@example.com>

gpg> save
```

# Exporting your key material
If you plan to use your GPG key on multiple workstations, or you need to share your public key with others, you can use the following commands:
```
gpg --armor --output you_pub.asc --export you@example.com
```

# Adding a new GPG key to your GitHub account
You will need to sign into GitHub to add your public key information. This will enable GitHub to verify that your signatures are in fact yours.

Copy the entire text block, including the ----BEGIN/END---- lines

Navigate to your GitHub’s account settings. Select SSH and GPG keys on the left hand navigation. Finally, click “New GPG key” and paste the above block into the input form. Give the key a name and save it.

Reference: [Adding a new GPG key to your GitHub account](https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification/adding-a-new-gpg-key-to-your-github-account)

# Setup git to sign commits and tags
Your commits should be signed with a Yubikey generated GPG key.

## Find your key handle
Begin by listing your GPG key with the LONG key format:
```
> gpg --list-secret-keys --keyid-format LONG <youremail@example.com>

C:/Users/youruser/AppData/Roaming/gnupg/pubring.kbx
-----------------------------------------------
sec>  rsa4096/36264D8005D951D8 2019-06-19 [SC]
      12341C42734692704224266256EDCD8005D9ABD3
      Card serial no. = 0006 12345678
uid                 [ultimate] Employee <employee@example.com>
ssb>  rsa4096/B4E1273375AC2412 2019-06-19 [S] [expires: 2021-06-18]
ssb>  rsa4096/2B5F29BB2DCA942D 2019-06-19 [E] [expires: 2021-06-18]
ssb>  rsa4096/AC59547D0CCB5ACE 2019-06-19 [A] [expires: 2021-06-18]
```

Determine the key handle for your signing key. This is the hexadecimal number on the line designated [S] above (B4E1273375AC2412).

## Configure git
Next, instruct Git that you want to use this key for signing:
```
git config --global user.signingKey B4E1273375AC2412
```
Consider turning auto signing on:
```
git config --global commit.gpgsign true
```
Otherwise, you will need to specify -S as an extra command line argument to git commit.

You will sometimes need to specify which GPG executable Git should use.

**Linux**
```
> git config --global gpg.program gpg2
```
**Windows**
```
> git config --global gpg.program "C:\\Program Files (x86)\\GnuPG\\bin\\gpg.exe"
```
**macOS**
```
> git config --global gpg.program gpg
```

# Signing commits
From a security standpoint, by default, Git doesn’t provide any assurance of authorship. Although every Git "blob" is hashed using SHA-1, this is only useful as an integrity check, i.e., to guarantee that the files and the commits that you are working with, are the exact same things they were when they were first created.

Git does provide the ability to sign your work. This allows users to verify that data is coming from a trusted source. Use the `-S` switch to sign your commits
```
git commit -S -m 'Fixed a small undocumented feature that made foo crash'
```

You will be prompted for your User PIN and the signed commit will be created. Note that the command shown above uses the capital letter S (the extended form would be `--gpg-sign`). Using the lowercase letter s will only include the text `Signed-off-by: Committer Name <committer@example.com>` in your commit message and **not** actually sign the commit.

To display the signature of the last commit you can use
```
git cat-file -p HEAD
tree c09dec94a1b2f8c4792fd0faef35623e0463fc73
parent 3fe8b3b4b9394678aeadfa4113e8982802f759f8
author Committer Name <committer@example.com> 1393232400 +0200
gpgsig -----BEGIN PGP SIGNATURE-----
 Version: GnuPG v1

iQEcBAABCAAGBQJW/PEoAAoJEJDLBFvTmUcBo58H/1hb+uhqVCRRFnQDJ7gHM+v1
6vgWxtaEpf86foJe+V/8r2dij2fKAPcbMQbeakfO0PplSRUY6+XnvXY+2uFHs2TB
BxsAz1HYLnl6jXRKpLqduqJLmnwnkwaMCr1Bx/rZ1CWAsKtwBf4AGpW7ws9Dv6zh
Y7EPcVeO4dvftTqCsoOu6ZBmw9U24DA5XCl7ZG2nDiW9spS8CTlznGA3/LJ56mWF
Rm+xaJbfFwr2KS5wdyZkzdEh0sIcbmAYVhnKkj4HiBegrK+wCcayOfc0YMzOUPL9
uJ4pB32g0jLJbpNHRXqhQ/OU9eCRG3B55UBpimvLOLok3si6d/fYd3zTmB9bJaE=
=Bh19
-----END PGP SIGNATURE-----
```

# Verifying commits
Signed commits can be verified in many different places. One way is to manually display the commit.
```
$ git show HEAD --show-signature

commit 552b36ec86790bfdac679ab23e6d61133ff0b383
gpg: Signature made Sat 22 Feb 2014 11:00:00 CEST using RSA key ID AABBCCDD
gpg: Good signature from "Committer Name <committer@example.com>"
Author: Committer Name <committer@example.com>
Date:   Sat Feb 22 11:00:00 2014 +0200

    Fixed a small undocumented feature that made foo crash
```

GPG has to have the public key of the signer to successfully verify the signature.

The previous command assumes that the commit of interest was the very last one. To verify a generic commit replace HEAD with the commit ID (`552b36ec86790bfdac679ab23e6d61133ff0b383` in this case).

Alternative commands to verify commit signatures are
```
git log --show-signature # Displays all commits and verify signed ones
```
```
git verify-commit HEAD # Displays and verify the latest commit
```

Reference: [GitHub signing commits](https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification/signing-commits)
 
# Signing tags
Tags are another one of the things that can be signed with Git. To do so you can use the `-s` switch
```
git tag foo-1.0 -s -m 'Release 1.0 of Foo'
 ```
After issuing the command, you will be prompted for your GPG User PIN and a signed tag will be created. You can check the result of this operation by running the following command
```
git show foo-1.0

tag foo-1.0
Tagger: Committer Name <committer@example.com>
Date:   Sat Feb 22 10:30:00 2014 +0200

Release 1.0 of Foo
-----BEGIN PGP SIGNATURE-----
Version: GnuPG v1

iQEcBAABCAAGBQJW/OhLAAoJEJDLBFvTmUcBKXAH/0i3O/F+YjD8xMknsMZGSa2/
/uGNnF5SUCxQjztWCJecHmp88GdyagT9rcgv/q6eniElwp3M3dQXBTdJ+tPH+m7G
yZdrmuLqrn/NTzZKj3E5xMT9IXJ+jg4RsfhALGqnrG5XFtsB5VVucURbEsrqNM+Y
k5PJPQD4jroT/jOOWBysQMlJRNVZGYhtCC2DkRPQo8lII8/KW5mGu/GJzpQepW4K
vnqd6h9vwhTddzQ+EosNGscQvQBM4+CtLznK3iCYEnDe111wCtMm/ukxd7378/tj
O+mdC0Q+mxTOgIHcgZKBFzVosxiSHVXo7cvmGgk8kuONdaGo2D0k0PqceZPOjRw=
=4ZAu
-----END PGP SIGNATURE-----
 ```

A similar output can also be achieved with the command
```
git cat-file -p foo-1.0
```

or with the command
```
git verify-tag foo-1.0
```

# Verifying tags
Once a tag has been signed, it is possible to ask Git to verify a signature for us. This is done by using the `-v` switch on the tag we want to verify
```
git tag -v foo-1.0

object 3fe8b3b4b9394678aeadfa4113e8982802f759f8
type commit
tag foo-1.0
tagger Committer Name <committer@example.com> 1393230600 +0200

Release 1.0 of Foo
gpg: Signature made Sat Feb 22 10:30:00 2014 CEST using RSA key ID AABBCCDD
gpg: Good signature from "Committer Name <committer@example.com>"
 ```

Keep in mind that, behind the scenes, this is invoking GPG which, in order to verify the signature for you, should be informed of who is the owner of key ID `AABBCCDD` by importing their public key. If this information is missing, you will receive an error message.

Reference: [GitHub signing tags](https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification/signing-tags)

# Merging and Pushing
When merging branches or tags, it is possible to ask Git to verify the signature of the commits being merged. This is done with
```
git merge --verify-signatures other_branch
 ```

If the signatures can not be verified, the merge will be aborted.
Similarly, the `-S` switch can be used to sign the commit resulting from a merge.

Also, if you created annotated tags, when you merge them Git will create a new commit for you. During this process it will also verify the involved signatures and include the verification output in the comment of the commit message.

Since Git version 2.2.0 it is also possible to sign git pushes by doing `git push --signed`. This is used to prove the intention the author had of pushing a specific set of commits and have them become the new tip of some branch.

# Additional Links
* [GitHub Managing commit signature verification](https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification)
* [Yubico PGP Walk-Through](https://developers.yubico.com/PGP/PGP_Walk-Through.html)
* [Yubico Git Signing](https://developers.yubico.com/PGP/Git_signing.html)


