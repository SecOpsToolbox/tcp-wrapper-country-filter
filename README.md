<h1 align="center">
	<a href="https://github.com/WolfSoftware">
		<img src="https://raw.githubusercontent.com/WolfSoftware/branding/master/images/general/banners/64/black-and-white.png " alt="Wolf Software Logo" />
	</a>
	<br>
	TCP wrapper to filter by country code
</h1>


<p align="center">
	<a href="https://travis-ci.com/SecOpsToolbox/tcp-wrapper-country-filter">
		<img src="https://img.shields.io/travis/com/SecOpsToolbox/tcp-wrapper-country-filter/master?style=for-the-badge&logo=travis" alt="Build Status">
	</a>
	<a href="https://github.com/SecOpsToolbox/tcp-wrapper-country-filter/releases/latest">
		<img src="https://img.shields.io/github/v/release/SecOpsToolbox/tcp-wrapper-country-filter?color=blue&style=for-the-badge&logo=github&logoColor=white&label=Latest%20Release" alt="Release">
	</a>
	<a href="https://github.com/SecOpsToolbox/tcp-wrapper-country-filter/commits/">
		<img src="https://img.shields.io/github/commits-since/SecOpsToolbox/tcp-wrapper-country-filter/latest.svg?style=for-the-badge&logo=github&logoColor=white" alt="Commits since release">
	</a>
	<a href="LICENSE.md">
		<img src="https://img.shields.io/badge/license-MIT-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" alt="Software License">
	</a>
	<br>
	<a href=".github/CODE_OF_CONDUCT.md">
		<img src="https://img.shields.io/badge/Code%20of%20Conduct-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" />
	</a>
	<a href=".github/CONTRIBUTING.md">
		<img src="https://img.shields.io/badge/Contributing-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" />
	</a>
	<a href=".github/SECURITY.md">
		<img src="https://img.shields.io/badge/Report%20Security%20Concern-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" />
	</a>
	<a href=".github/SUPPORT.md">
		<img src="https://img.shields.io/badge/Get%20Support-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" />
	</a>
</p>

## Overview

This is a [TCP wrapper](https://en.wikipedia.org/wiki/TCP_Wrappers) which will filter server connection attempts based on the country of origin. It can be configured in one of two different ways:

1. Allow connections only from a specified list of countries.
1. Deny connections EXCEPT those from a specified list of countries.

This allows for the dynamic blocking (or allowing) of an entire country without having to manage or maintain IP lists, which can often be very large.

* USA has over 1.5 billion IP addresses spanning approximately 450,000 different blocks.
* China has over 300 million IP addresses spanning approximately 4,000 different blocks.
* Russia has over 40 million IP addresses spanning approximately 10,000 different blocks.

> The solution is only as accurate as the GeoIP database, however most tools for identifying a country from an IP are at least 99% accurate.

### Security

The use of TCP wrappers does not eliminate the need for a properly configured firewall. This script should be seen as **part** of your security solution, **not** the whole of it.

### Prerequisites

This tool relies on **geoiplookup**, if it is not installed then the script will log an error and **allow** the connection, even if the default action is **DENY**. The reason for this is that without this **ALL** connections would be blocked including your own *(which would be bad)*.

#### Installing the Prerequisites

> This may require additional apt or yum sources depending on your distribution.

<b>Debian / Ubuntu</b>

```shell
# apt-get install geoip-bin geoip-database
```

<b>CentOS / RHEL</b>

```shell
# yum install GeoIP GeoIP-data
```

By default this will install the *free* version of the GeoLite Country binary database (GeoIP.dat etc.), usually in the /usr/local/share or /usr/share directory. The specific location doesn't matter as the geoiplookup command will know where to look for the data files.

> We currently do not support GeoIP2 format (mmdb) or automated updates from MaxMind, although is this on the roadmap for this tool.

#### Testing the Prerequisites

Look up one of Google’s IPs.

```shell
# geoiplookup 74.125.225.33
GeoIP Country Edition: US, United States
```

> If you see the above or similar then geoiplookup is installed and working.

### Configuration

Although this was developed for use with sshd, the principle should work for any service that is supported by TCP wrappers, however in this documentation we will use sshd.

#### Install the filter

Copy the [script](src/country-filter.sh) to /usr/local/sbin/country-filter (and ensure that it is executable [*chmod +x]*).

Out of the box the country list is empty and the script has the default [`ACTION`](src/country-filter.sh#L28) of `DENY` (only block countries in the list), so the net effect at this point is to block nothing.

#### Adding countries

To add countries to the list, add them to the [`COUNTRIES`](src/country-filter.sh#L25) variable. This is a space separated list of country codes (2 letter codes). Example [country code list](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) from Wikipedia.

There are times where a country cannot be identified, if you want to block all entries where a country cannot be identified, add `XX` to the [`COUNTRIES`](src/country-filter.sh#L25) variable.

#### Allow or Deny

By default the script will deny connections from any country listed in the [`COUNTRIES`](src/country-filter.sh#L25) variable, however you can invert this logic and only allow connections from these countries, by setting the [`ACTION`](src/country-filter.sh#L28) variable to `ALLOW`.

If you change the default [`ACTION`](src/country-filter.sh#L28) to `ALLOW`, ensure your own country is in the list of countries before you do this, otherwise you will no longer be able to connect to your server. This won't effect existing open connections, so test with a new connection attempt to ensure the configuration is correct.

#### Process Ordering

In Linux/Unix based systems the processing order for TCP wrappers is as follows:

1. hosts.allow
2. hosts.deny

This means that anything that is not handled (allowed / denied) by hosts.allow will be handled by hosts.deny.

#### /etc/hosts.allow

The following configuration will tell the system to pass all IPs, for ssh connections, to the country-filter. The return code of the filter specifies the action to be taken.

1. 0 = Success - allow the connection.
2. 1 = Failure - deny the connection.

```shell
sshd: ALL: aclexec /usr/local/sbin/country-filter %a 
```

> aclexec tells the system to execute the following script and %a is replace by the current IP address.

#### /etc/hosts.deny

The following configuration will tell the system to deny all ssh connections. 

```shell
sshd: ALL
```

> This should never be reached because all cases should be handled by the country filter, but as with all security configurations **protection in depth** is key and having a safe / secure fallback position is preferable.

## Alternatives

We provide a number of different [TCP Wrapper filters](https://github.com/SecOpsToolbox?q=in%3Aname+tcp+wrapper+filter&type=&language=).

## Multiple Rules

If you wish to use more than one of our TCP Wrappers then please refer to our [TCP Wrapper Multiplexer](https://github.com/SecOpsToolbox/tcp-wrapper-multiplexer).

## Contributors

<p>
	<a href="https://github.com/TGWolf">
		<img src="https://img.shields.io/badge/Wolf-black?style=for-the-badge" />
	</a>
</p>

## Show Support

<p>
	<a href="https://ko-fi.com/wolfsoftware">
		<img src="https://img.shields.io/badge/Ko%20Fi-blue?style=for-the-badge&logo=ko-fi&logoColor=white" />
	</a>
</p>
