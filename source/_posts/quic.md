---
title: quic
date: 2022-03-12 23:02:48
tags: live
categories: protocol
---


Internet Engineering Task Force (IETF)                   J. Iyengar, Ed.
Request for Comments: 9000                                        Fastly
Category: Standards Track                                M. Thomson, Ed.
ISSN: 2070-1721                                                  Mozilla
                                                                May 2021


           QUIC: A UDP-Based Multiplexed and Secure Transport

### Abstract

   This document defines the core of the QUIC transport protocol.  QUIC
   provides applications with flow-controlled streams for structured
   communication, low-latency connection establishment, and network path
   migration.  QUIC includes security measures that ensure
   confidentiality, integrity, and availability in a range of deployment
   circumstances.  Accompanying documents describe the integration of
   TLS for key negotiation, loss detection, and an exemplary congestion
   control algorithm.
   本文档定义了 QUIC 传输协议的核心。
   QUIC 为应用程序提供流控流，用于结构化通信、低延迟连接建立和网络路径迁移。
   QUIC 包括在一系列部署环境中确保机密性、完整性和可用性的安全措施。
   随附的文件描述了用于密钥协商、丢失检测和示例性拥塞控制算法的 TLS 集成。
