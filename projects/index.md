---
title: Projects
layout: page
excerpt: "A list of open-source projects I've created or heavily contributed to."
tags: [priestjim, omnilectual, projects, github]
---

Below is a non exhaustive list of projects I've published from time to time.

## Erlang Projects

* [**gen_rpc**](https://github.com/priestjim/gen_rpc): An out-of-band RPC library written in Erlang. Combines the safety of the `rpc` library with the versatility of remote spawn. Uses concurrent TCP channels for each interconnected node, overcomming single-mailbox issues in high-load environments.

## Elixir Projects

* [**exrpc**](https://github.com/priestjim/exrpc): An Elixir port of the `gen_rpc` project above. Built it to both get exposure to the language and to provide a more "native" port of `gen_rpc` to Elixir. `ExRPC` and `gen_rpc` nodes are transparently interoperable.

## Chef and Automation

* [**chef-openresty**](https://github.com/priestjim/chef-openresty): The official Chef cookbook for CloudFlare's NGINX superbundle. Contains a lot of NGINX configuration optimizations out-of-the-box, automatic CPU affinity calculation, security configuration and OS-specific optimizations.

* [**chef-rundeck**](https://github.com/priestjim/chef-rundeck): The official Chef cookbook for the Rundeck administration console. Contains a couple of LWRPs and supports both Debian and CentOS installations.

* [**chef-phpmyadmin**](https://github.com/priestjim/chef-phpmyadmin): The official Chef cookbook for PHPMyAdmin. Supports automatic PHP-FPM and NGINX configuration and offers LWRPs for database server profile creation and PMA database provisioning.

* [**chef-pcre**](https://github.com/priestjim/chef-pcre): The official Chef cookbook for the PCRE library.

* [**chef-jemalloc**](https://github.com/priestjim/chef-jemalloc): The official Chef cookbook for the jemalloc library.

* [**chef-php**](https://github.com/priestjim/chef-php): A fork of the official PHP community cookbook that diverged too much from the community version to get merged back in. Offers the `php-fpm` LWRP.

* [**chef-azure**](https://github.com/priestjim/chef-azure): A rogue cookbook for Linux VMs running on Microsoft Azure. The cookbook provides the `azure` OHAI plugin that provides heuristic detection of AWS-like properties such as instance type, region etc.

* [**knife-tagbulk**](https://github.com/priestjim/knife-tagbulk): A `knife` plugin in the form of a gem that allows creation and deletion of tags en-masse, using standard Chef queries.

## PHP Projects

* [**spamshield**](https://github.com/priestjim/spamshield): A neat PHP script that scans the Postfix mail queue and produces nice reports and notifications about possible spam issues.

* [**relaxx**](https://github.com/priestjim/relaxx): A fork of the well-known Relaxx player with vast protocol support and UI improvements.
