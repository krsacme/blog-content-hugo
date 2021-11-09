---
author:
  DisplayName: Saravanan KR
  Link: krsacme
date: "2020-11-12T09:00:00+05:30"
header:
  caption: ""
  image: ""
highlight: true
math: false
tags:
- nfv
- 4g
- epc
title: 4G Architecture - Basic
type: post
---

This is a post about the basic building blocks of the NFV 4G architecture. The
idea is to have a simpler explantions about 4G and dive deeper in to enhanced
versions and then to 5G and to edge. This post does not explain all the
modules, but captures the important ones to explain the end-to-end usecase.

The basic layers of a NFV communication are:

* User Equipment (UE)
* Core Network

User Equipment
--------------

UE will be attached in a Radio Access Network to establish a connection. In
4G, all the communication will be based on packets. UE should be capable of
handling 4G signals in order to communicate with RAN. The communication will
be use the 4G spectrum (allocated to a provider) and format defined by (3GPP)
