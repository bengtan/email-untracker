# email-untracker

`email-untracker` (or Untracker for short) is a self-hosted CLI tool to investigate and undo tracking in email newsletters.

## Table of contents

* [Overview](#overview)
* [Benefits](#benefits)
* [Getting started: Try it on an email](#getting-started-try-it-on-an-email)
  * [.eml files](#eml-files)
  * [Gmail](#gmail)
  * [Postfix / Dovecot / Maildir](#postfix--dovecot--maildir)
* [Usage (CLI)](#usage-cli)
  * [parse](#parse)
  * [bounce](#bounce)
  * [deliver](#deliver)
* [Usage (email server)](#usage-email-server)
  * [Setup](#setup)
  * [Usage](#usage)
  * [Forwarding loops](#forwarding-loops)
* [Building](#building)
* [How it works](#how-it-works)
* [SaaS version](#saas-version)
* [Sponsorware and licensing](#sponsorware-and-licensing)
  * [Dual licensing](#dual-licensing)
* [New features](#new-features)
* [Open source, not open contribution](#open-source-not-open-contribution)
* [Background reading](#background-reading)

## Overview

`email-untracker` (or Untracker for short) is a self-hosted CLI tool to investigate and undo tracking in email newsletters.

* Remove tracking links
* Remove tracking/spy pixels
* Add-on to your self-hosted email server
* Open source
* Completely private

Although Untracker is written to untrack newsletters, a future version might be able to untrack all emails.

## Benefits

* Better user privacy
* Email newsletters are more user friendly
* Helps detect phishing attacks
* Plain-text emails are readable again

Note that Untracker doesn't magically bypass the tracking links (but see [New Features](#new-features) below). Instead, it triggers all of them from your mail server as a one-time occurrence.

Hence, it’s privacy by obfuscation. But if your mail server is in a data centre somewhere, this does prevent your web browser/IP address/location from being leaked.

And the tracking is only triggered once instead of everytime you click.

## Getting started: Try it on an email

Choose a recent [release](https://github.com/bengtan/email-untracker/releases) and download the appropriate binary to your computer. Rename it to `email-untracker`.

Then run:

```
email-untracker parse -i newsletter.eml -v -h
```

where `newsletter.eml` is a file containing an email newsletter.

Untracker will then:

* Print out links in the email's HTML body. If a link is a tracking link, crawl the link and print out the destination URL.
* Print out any tracking pixels

The `parse` command parses the file specified by `-i`. `-v` tells Untracker to be verbose. `-h` tells Untracker to process the HTML body.

### .eml files

An .eml file is an email in it's raw form. It is a verbatim representation of the email as it is passed around between email servers or from email server to a mail delivery agent (MDA).

How to get an .eml file depends on your email setup but here are methods for two common situations.

### Gmail

* Open an email for viewing.
* Select the 'More' menu (ie. three vertically-stacked dots) to display additional options. One of the options is 'Show original'.
* Click on 'Show original'.
* On the page that results, click on 'Download original'.
* Download the file and save it to your computer.

### Postfix / Dovecot / Maildir

If you have access to your mail server and it stores email in [Maildir](https://en.wikipedia.org/wiki/Maildir) format (which Postfix/Dovecot does), then each email is already an .eml file.

* Go to your home directory and look for a directory called `Maildir`.
* Look for a subdirectory called `new` or `cur`. Files in these subdirectories are already in .eml format (even though they don't have an .eml extension).

You may have to do some searching/grepping to find a file which is an email newsletter.

## Usage (CLI)

Untracker has two modes: `parse` and `bounce`

(In the future, there may be a mode `deliver` which delivers to a mailbox.)

### parse

This mode parses an email.

It is useful for debugging or investigation. The email body (plain-text or HTML or both, original or processed) can be saved to a file.

```
// Print original plain-text body to original.txt
email-untracker parse -i newsletter.eml -P -o original.txt

// Print processed plain-text body to untracked.txt
email-untracker parse -i newsletter.eml -p -o untracked.txt

// Print original HTML body to original.html
email-untracker parse -i newsletter.eml -H -o original.html

// Print processed HTML body to untracked.html
email-untracker parse -i newsletter.eml -h -o untracked.html

// Like previously
// Verbose. Process a maximum of three links
email-untracker parse -i newsletter.eml -h -o untracked.html -v -n3
```

If `-i` isn't specified, then it reads from stdin.

```
// Print original plain-text body to stdout
cat newsletter.eml | email-untracker parse -P -o -
```

### bounce

This mode processes/untracks an email and sends it (via SMTP on localhost:587).

```
// Process an email and send it back to the sender.
// Warning: This will actually send an email
email-untracker bounce -f from@example.com -i newsletter.eml

// Like previously but verbose so you can see the processing
// Warning: This will actually send an email
email-untracker bounce -f from@example.com -i newsletter.eml -v

// Like previously but send it to email@example.com instead
// Warning: This will actually send an email
email-untracker bounce -f from@example.com -i newsletter.eml -v -t email@example.com
```

If `-i` isn't specified, then it reads from stdin.

```
// Process an email and send it back to the sender.
// Warning: This will actually send an email
cat newsletter.eml | email-untracker bounce
```

### deliver

This is a proposed future feature.

In this mode, Untracker processes an email and delivers to a local (Maildir?) mailbox instead of sending it.

## Usage (email server)

(This section presumes that you self-host your email server and it is running Postfix/Dovecot or similar. This has been tested on Ubuntu 18.04 and it should work on most linux systems since the Mail Delivery Agent specification hasn't changed much.)

Untracker can be integrated into an email server by plugging it in as a Mail Delivery Agent. When an email is received, Untracker is called to handle it (the email is passed via stdin). Instead of delivering the email to a local mailbox, Untracker processes it and sends it off instead.

This uses Untracker in bounce mode.

### Setup

* Copy the binary to the email server (ie. `/usr/bin/email-untracker`)
* Create a new user account (ie. `untracker`). Note it's email address (ie. `untracker@example.com`).
* In the user's home directory, create a file `.forward` with the following two lines:

```
\untracker
"|/usr/bin/email-untracker bounce -f untracker@example.com"
```

* Ensure `.forward` is accessible only by the user:

```
chown untracker .forward
chmod 600 .forward
```

The first line in `.forward` tells the mail system to deliver email to the account normally. The backslash informs the mail system not to process `.forward` for this delivery.

If the first line is omitted, then emails are not delivered ie. they won't be stored.

The second line in `.forward` tells the mail system to call `email-untracker` (with the email in stdin). Untracker then processes the email and sends it off.

### Usage

To use Untracker, forward an email to the account's email address (ie. `untracker@example.com`). Untracker will untrack the email and send it back.

Note that you will want to keep the email address (ie. `untracker@example.com`) secret since it acts like an open relay and is vulnerable to spam.

You can also set up automatic forwarding of your newsletters to `untracker@example.com`. How you do this will depend on your email system.

### Forwarding loops

If you set up automatic forwarding, configure it so that already processed emails are not forwarded again. Otherwise, a forwarding loop could result.

For example, forwarding could be omitted if an email has '[untracker]' in the subject field.

Untracker will warn about forwarding loops by sending back an email with the message 'Infinite loop detected.'

## Building

Untracker is written in GO. Install GO on your system. Then:

```
go build
```

will compile a binary.

## How it works

Please read the [original article](https://bengtan.com/blog/email-cleaner-clean-tracking-links-and-pixels/#how-it-works).

Note that Untracker currently only supports untracking newsletters from three major mailing list providers (MailChimp, ConvertKit, Substack). I’d happily add more. Just let me know.

## SaaS version

If you don't have access to your mail server, you can use the [original Email Cleaner](https://bengtan.com/blog/email-cleaner-clean-tracking-links-and-pixels/) service (which is powered by Untracker). Please note that you’d be disclosing your email newsletters to a third party.

## Sponsorware and licensing

Untracker is [sponsorware](https://github.com/sponsorware/docs).

There are two repositories:

* A free, publicly available [repository](https://github.com/bengtan/email-untracker) licensed under the [Affero General Public License Version 3](https://www.gnu.org/licenses/agpl-3.0.en.html) (AGPL) open source license, and 
* A sponsors-only [repository](https://github.com/bengtan/email-untracker-sponsorware) licensed under the [PolyForm Internal Use License 1.0.0](https://polyformproject.org/licenses/internal-use/1.0.0/) 'source available' license.

At the beginning:

* The free repository contains binaries.
* The sponsors-only repository contains source code.

Once 75 sponsors have been reached:

* The current source code is open-sourced and published to the free repository.

From there-on:

* Ongoing development and new features occurs in the sponsors-only repository.
* (If there are 75 sponsors or more) When new versions or features are released, the source code of the old versions are published to the free repository.

### Dual licensing

If you wish to use Untracker under different license terms, please contact me.

## New features

This is a speculative list. Whether these features eventuate will depend on sponsorships and other things.

* Bypass tracking entirely &mdash; Some mailing list providers encode the destination URL in the tracking link itself. For these cases, Untracker can decode the URL without crawling, thus bypassing tracking entirely.
* Deliver mode &mdash; Instead of bouncing a processed email, deliver it to the original mailbox.
* Ability to untrack all emails &mdash; Requires deliver mode.
* Convert emails to plain text (or rather, just strip out the HTML part) — The ultimate in privacy.
* Crawl images and embed them into the email. This can stop image-based tracking for all images, not just spy pixels.
* ‘Deep inspection’ mode &mdash;  Crawls all links and images and report any which are trackers. Useful for detecting new, unknown trackers.
* Crawling via a HTTP proxy (or tor?)
* Coverage of more mailing list providers and tracking pixels.

## Open source, not open contribution

I’ve currently decided not to accept source code patches. This is for a number of reasons.

I can restrain creeping featurism and keep the code tight, thus reducing long term maintenance burden and potential for burn-out.

It avoids any ambiguity over copyright of Untracker. I would like to retain copyright over it.

Some FLOSS contributors get upset when open source software (which has some community-contributed patches) is monetised or relicensed under a different software license. I don’t have an opinion on this argument but I’d like to avoid it entirely.

I hope I don’t sound unfriendly about this. I’d be happy for the community to get involved and post bug reports and feature requests. These activities are valuable too. Software development isn’t just coding.

If you do submit source code patches, I’d happily look at them but I’d rewrite them from scratch.

## Background reading

https://bengtan.com/tags/untracker/
