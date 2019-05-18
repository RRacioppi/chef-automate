# chef-automate

## Description

Requirements: 

- Vagrant
- Image bento/ubuntu-16.04

This repository provides a Vagrantfile that provisions a sample architecture for Chef Automate. The cluster is composed of three Virtual Machines: 

- Chef Node
- Chef Automate
- Chef Server (Its provisioner automatically integrates Chef Server with Chef Automate. For simplicity, Chef Server acts as a Chef Workstation in order to bootstrap Chef Node)

## Usage

1. cd to Vagrantfile's directory

2. run `$ vagrant up`
