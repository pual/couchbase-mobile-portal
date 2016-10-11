---
id: upgrade
title: Upgrade
permalink: ready/training/deploy/upgrade/index.html
---

In this lesson you'll learn how to install upgrades for Sync Gateway and Couchbase Server with zero downtime.

#### Requirements

One instance with the following:

- Ubuntu >= 12.04, =< 14.04
- RAM >= 2GB

## Architecture

## Upgrade scenario

In this lesson you will continue where you left off in the previous lesson and add another Sync Gateway node.

## Resiliency

In this section you will add a Sync Gateway node while you perform the upgrade.

ADD NGINX?

### Try it out

1. 

## Taking the database offline

It is recommended to take the database offline before performing the upgrade. Interrupting a replication session that is currently in progress might result in specific documents not being pushed or pulled, but it will not result in data loss.

### Try it out

1. Take it offline

## Upgrading the node

## Conclusion