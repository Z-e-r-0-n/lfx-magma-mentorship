# Magma Deployment Report

## 1. Objective

This report documents the process of deploying Magma using Docker on a virtual machine as part of the LFX Magma mentorship task.

## 2. Environment

## 2. Environment

- Host machine: mediaserver3
- Host OS: Ubuntu 26.04 LTS
- VM software: KVM/QEMU with libvirt
- VM name: magma-ubuntu20
- VM OS: Ubuntu 20.04.6 LTS
- VM CPU allocation: 6 vCPUs
- VM RAM: 16 GB
- VM disk: 120 GB qcow2, root filesystem ~117 GB
- Network mode: libvirt default NAT network
- VM IP: 192.168.122.146
- Docker version: To be installed
- Docker Compose version: To be installed
- Magma repository/branch: To be cloned

## 3. Setup Steps

### 3.1 VM Preparation

### 3.1 VM Preparation

A clean Ubuntu 20.04 VM was created on the mediaserver3 host using KVM/QEMU and libvirt. The VM was created from the Ubuntu 20.04 cloud image and configured using cloud-init.

Commands/tools used:
- qemu-system-x86
- libvirt
- virt-install
- cloud-localds
- Ubuntu 20.04 focal cloud image

The VM was allocated 6 vCPUs, 16 GB RAM, and a 120 GB qcow2 disk. It received IP address 192.168.122.146 on the default libvirt NAT network.

### 3.2 Docker Installation

_To be filled during setup._

### 3.3 Magma Repository Setup

_To be filled during setup._

### 3.4 Magma Deployment Commands

_To be filled during setup._

## 4. Issues Encountered

| Issue | Cause | Resolution |
|---|---|---|
| | | |

## 5. Verification

_To be filled after deployment._

## 6. Logs/Screenshots

_To be filled with relevant logs or screenshot references._

## 7. Final Status

_To be filled after setup._
