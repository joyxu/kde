

#### [PATCH v1 0/8] vfio: expose virtual Shared Virtual Addressing to VMs
##### From: "Liu, Yi L" <yi.l.liu@intel.com>
From: Liu Yi L <yi.l.liu@intel.com>

```c
From patchwork Sun Mar 22 12:31:57 2020
Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
X-Patchwork-Submitter: Liu Yi L <yi.l.liu@intel.com>
X-Patchwork-Id: 11451675
Return-Path: <SRS0=VAWz=5H=vger.kernel.org=kvm-owner@kernel.org>
Received: from mail.kernel.org (pdx-korg-mail-1.web.codeaurora.org
 [172.30.200.123])
	by pdx-korg-patchwork-2.web.codeaurora.org (Postfix) with ESMTP id 910E6139A
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 22 Mar 2020 12:26:53 +0000 (UTC)
Received: from vger.kernel.org (vger.kernel.org [209.132.180.67])
	by mail.kernel.org (Postfix) with ESMTP id 7A1E4207FF
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 22 Mar 2020 12:26:53 +0000 (UTC)
Received: (majordomo@vger.kernel.org) by vger.kernel.org via listexpand
        id S1727212AbgCVM0x (ORCPT
        <rfc822;patchwork-kvm@patchwork.kernel.org>);
        Sun, 22 Mar 2020 08:26:53 -0400
Received: from mga18.intel.com ([134.134.136.126]:51562 "EHLO mga18.intel.com"
        rhost-flags-OK-OK-OK-OK) by vger.kernel.org with ESMTP
        id S1726984AbgCVM0Z (ORCPT <rfc822;kvm@vger.kernel.org>);
        Sun, 22 Mar 2020 08:26:25 -0400
IronPort-SDR: 
 VbsgSx7gSsGf0N6hgS7G+ORwFbJCKjK9GL+4sY/Na9D3xsa/K+iMMK0VegOFi7XmAvrjF+u+tB
 YrsqFxuW5C5g==
X-Amp-Result: SKIPPED(no attachment in message)
X-Amp-File-Uploaded: False
Received: from orsmga008.jf.intel.com ([10.7.209.65])
  by orsmga106.jf.intel.com with ESMTP/TLS/ECDHE-RSA-AES256-GCM-SHA384;
 22 Mar 2020 05:26:23 -0700
IronPort-SDR: 
 Cy6rHL7Yt8Bx00K+whxJzVWM95dDmNBC+s3FORZdpg1GwjHmSDiRC2tikfoh+bCYeuAvqfteE8
 Zdeyt14VdXSw==
X-ExtLoop1: 1
X-IronPort-AV: E=Sophos;i="5.72,292,1580803200";
   d="scan'208";a="239663862"
Received: from jacob-builder.jf.intel.com ([10.7.199.155])
  by orsmga008.jf.intel.com with ESMTP; 22 Mar 2020 05:26:23 -0700
From: "Liu, Yi L" <yi.l.liu@intel.com>
To: alex.williamson@redhat.com, eric.auger@redhat.com
Cc: kevin.tian@intel.com, jacob.jun.pan@linux.intel.com,
        joro@8bytes.org, ashok.raj@intel.com, yi.l.liu@intel.com,
        jun.j.tian@intel.com, yi.y.sun@intel.com, jean-philippe@linaro.org,
        peterx@redhat.com, iommu@lists.linux-foundation.org,
        kvm@vger.kernel.org, linux-kernel@vger.kernel.org, hao.wu@intel.com
Subject: [PATCH v1 0/8] vfio: expose virtual Shared Virtual Addressing to VMs
Date: Sun, 22 Mar 2020 05:31:57 -0700
Message-Id: <1584880325-10561-1-git-send-email-yi.l.liu@intel.com>
X-Mailer: git-send-email 2.7.4
Sender: kvm-owner@vger.kernel.org
Precedence: bulk
List-ID: <kvm.vger.kernel.org>
X-Mailing-List: kvm@vger.kernel.org

From: Liu Yi L <yi.l.liu@intel.com>

Shared Virtual Addressing (SVA), a.k.a, Shared Virtual Memory (SVM) on
Intel platforms allows address space sharing between device DMA and
applications. SVA can reduce programming complexity and enhance security.

This VFIO series is intended to expose SVA usage to VMs. i.e. Sharing
guest application address space with passthru devices. This is called
vSVA in this series. The whole vSVA enabling requires QEMU/VFIO/IOMMU
changes. For IOMMU and QEMU changes, they are in separate series (listed
in the "Related series").

The high-level architecture for SVA virtualization is as below, the key
design of vSVA support is to utilize the dual-stage IOMMU translation (
also known as IOMMU nesting translation) capability in host IOMMU.


    .-------------.  .---------------------------.
    |   vIOMMU    |  | Guest process CR3, FL only|
    |             |  '---------------------------'
    .----------------/
    | PASID Entry |--- PASID cache flush -
    '-------------'                       |
    |             |                       V
    |             |                CR3 in GPA
    '-------------'
Guest
------| Shadow |--------------------------|--------
      v        v                          v
Host
    .-------------.  .----------------------.
    |   pIOMMU    |  | Bind FL for GVA-GPA  |
    |             |  '----------------------'
    .----------------/  |
    | PASID Entry |     V (Nested xlate)
    '----------------\.------------------------------.
    |             |   |SL for GPA-HPA, default domain|
    |             |   '------------------------------'
    '-------------'
Where:
 - FL = First level/stage one page tables
 - SL = Second level/stage two page tables

There are roughly four parts in this patchset which are
corresponding to the basic vSVA support for PCI device
assignment
 1. vfio support for PASID allocation and free for VMs
 2. vfio support for guest page table binding request from VMs
 3. vfio support for IOMMU cache invalidation from VMs
 4. vfio support for vSVA usage on IOMMU-backed mdevs

The complete vSVA kernel upstream patches are divided into three phases:
    1. Common APIs and PCI device direct assignment
    2. IOMMU-backed Mediated Device assignment
    3. Page Request Services (PRS) support

This patchset is aiming for the phase 1 and phase 2, and based on Jacob's
below series.
[PATCH V10 00/11] Nested Shared Virtual Address (SVA) VT-d support:
https://lkml.org/lkml/2020/3/20/1172

Complete set for current vSVA can be found in below branch.
https://github.com/luxis1999/linux-vsva.git: vsva-linux-5.6-rc6

The corresponding QEMU patch series is as below, complete QEMU set can be
found in below branch.
[PATCH v1 00/22] intel_iommu: expose Shared Virtual Addressing to VMs
complete QEMU set can be found in below link:
https://github.com/luxis1999/qemu.git: sva_vtd_v10_v1

Regards,
Yi Liu

Changelog:
	- RFC v1 -> Patch v1:
	  a) Address comments to the PASID request(alloc/free) path
	  b) Report PASID alloc/free availabitiy to user-space
	  c) Add a vfio_iommu_type1 parameter to support pasid quota tuning
	  d) Adjusted to latest ioasid code implementation. e.g. remove the
	     code for tracking the allocated PASIDs as latest ioasid code
	     will track it, VFIO could use ioasid_free_set() to free all
	     PASIDs.

	- RFC v2 -> v3:
	  a) Refine the whole patchset to fit the roughly parts in this series
	  b) Adds complete vfio PASID management framework. e.g. pasid alloc,
	  free, reclaim in VM crash/down and per-VM PASID quota to prevent
	  PASID abuse.
	  c) Adds IOMMU uAPI version check and page table format check to ensure
	  version compatibility and hardware compatibility.
	  d) Adds vSVA vfio support for IOMMU-backed mdevs.

	- RFC v1 -> v2:
	  Dropped vfio: VFIO_IOMMU_ATTACH/DETACH_PASID_TABLE.

Liu Yi L (8):
  vfio: Add VFIO_IOMMU_PASID_REQUEST(alloc/free)
  vfio/type1: Add vfio_iommu_type1 parameter for quota tuning
  vfio/type1: Report PASID alloc/free support to userspace
  vfio: Check nesting iommu uAPI version
  vfio/type1: Report 1st-level/stage-1 format to userspace
  vfio/type1: Bind guest page tables to host
  vfio/type1: Add VFIO_IOMMU_CACHE_INVALIDATE
  vfio/type1: Add vSVA support for IOMMU-backed mdevs

 drivers/vfio/vfio.c             | 136 +++++++++++++
 drivers/vfio/vfio_iommu_type1.c | 419 ++++++++++++++++++++++++++++++++++++++++
 include/linux/vfio.h            |  21 ++
 include/uapi/linux/vfio.h       | 127 ++++++++++++
 4 files changed, 703 insertions(+)
```
#### [PATCH v1 0/2] vfio/pci: expose device's PASID capability to VMs
##### From: "Liu, Yi L" <yi.l.liu@intel.com>
From: Liu Yi L <yi.l.liu@intel.com>

```c
From patchwork Sun Mar 22 12:33:12 2020
Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
X-Patchwork-Submitter: Liu Yi L <yi.l.liu@intel.com>
X-Patchwork-Id: 11451683
Return-Path: <SRS0=VAWz=5H=vger.kernel.org=kvm-owner@kernel.org>
Received: from mail.kernel.org (pdx-korg-mail-1.web.codeaurora.org
 [172.30.200.123])
	by pdx-korg-patchwork-2.web.codeaurora.org (Postfix) with ESMTP id 1A144913
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 22 Mar 2020 12:27:39 +0000 (UTC)
Received: from vger.kernel.org (vger.kernel.org [209.132.180.67])
	by mail.kernel.org (Postfix) with ESMTP id ED19120836
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 22 Mar 2020 12:27:38 +0000 (UTC)
Received: (majordomo@vger.kernel.org) by vger.kernel.org via listexpand
        id S1727118AbgCVM1d (ORCPT
        <rfc822;patchwork-kvm@patchwork.kernel.org>);
        Sun, 22 Mar 2020 08:27:33 -0400
Received: from mga14.intel.com ([192.55.52.115]:23953 "EHLO mga14.intel.com"
        rhost-flags-OK-OK-OK-OK) by vger.kernel.org with ESMTP
        id S1726756AbgCVM1d (ORCPT <rfc822;kvm@vger.kernel.org>);
        Sun, 22 Mar 2020 08:27:33 -0400
IronPort-SDR: 
 1AiDmzmXy41admJC3WnxMxIS/ZkYMsTrOZMxgVBkmhCwKekNUxOy8ogd0lAiWHolpCPsM6Sg2O
 WdnrTV4vNf7w==
X-Amp-Result: SKIPPED(no attachment in message)
X-Amp-File-Uploaded: False
Received: from fmsmga001.fm.intel.com ([10.253.24.23])
  by fmsmga103.fm.intel.com with ESMTP/TLS/ECDHE-RSA-AES256-GCM-SHA384;
 22 Mar 2020 05:27:32 -0700
IronPort-SDR: 
 UGOINlO9ScM6KwkGezIO4BGXLoADUcKXhv4X+IQHXWbULydX4TwDMp3wBEb7uD6gxfXNxwArZm
 ynlNw9EmimTQ==
X-ExtLoop1: 1
X-IronPort-AV: E=Sophos;i="5.72,292,1580803200";
   d="scan'208";a="356846582"
Received: from jacob-builder.jf.intel.com ([10.7.199.155])
  by fmsmga001.fm.intel.com with ESMTP; 22 Mar 2020 05:27:32 -0700
From: "Liu, Yi L" <yi.l.liu@intel.com>
To: alex.williamson@redhat.com, eric.auger@redhat.com
Cc: kevin.tian@intel.com, jacob.jun.pan@linux.intel.com,
        joro@8bytes.org, ashok.raj@intel.com, yi.l.liu@intel.com,
        jun.j.tian@intel.com, yi.y.sun@intel.com, jean-philippe@linaro.org,
        peterx@redhat.com, iommu@lists.linux-foundation.org,
        kvm@vger.kernel.org, linux-kernel@vger.kernel.org, hao.wu@intel.com
Subject: [PATCH v1 0/2] vfio/pci: expose device's PASID capability to VMs
Date: Sun, 22 Mar 2020 05:33:12 -0700
Message-Id: <1584880394-11184-1-git-send-email-yi.l.liu@intel.com>
X-Mailer: git-send-email 2.7.4
Sender: kvm-owner@vger.kernel.org
Precedence: bulk
List-ID: <kvm.vger.kernel.org>
X-Mailing-List: kvm@vger.kernel.org

From: Liu Yi L <yi.l.liu@intel.com>

Shared Virtual Addressing (SVA), a.k.a, Shared Virtual Memory (SVM) on
Intel platforms allows address space sharing between device DMA and
applications. SVA can reduce programming complexity and enhance security.

To enable SVA, device needs to have PASID capability, which is a key
capability for SVA. This patchset exposes the device's PASID capability
to guest instead of hiding it from guest.

The second patch emulates PASID capability for VFs (Virtual Function) since
VFs don't implement such capability per PCIe spec. This patch emulates such
capability and expose to VM if the capability is enabled in PF (Physical
Function).

However, there is an open for PASID emulation. If PF driver disables PASID
capability at runtime, then it may be an issue. e.g. PF should not disable
PASID capability if there is guest using this capability on any VF related
to this PF. To solve it, may need to introduce a generic communication
framework between vfio-pci driver and PF drivers. Please feel free to give
your suggestions on it.

Regards,
Yi Liu

Changelog:
	- RFC v1 -> Patch v1:
	  Add CONFIG_PCI_ATS #ifdef control to avoid compiling error.

Liu Yi L (2):
  vfio/pci: Expose PCIe PASID capability to guest
  vfio/pci: Emulate PASID/PRI capability for VFs

 drivers/vfio/pci/vfio_pci_config.c | 327 ++++++++++++++++++++++++++++++++++++-
 1 file changed, 324 insertions(+), 3 deletions(-)
```
#### [PATCH v1 00/22] intel_iommu: expose Shared Virtual Addressing to VMs
##### From: Liu Yi L <yi.l.liu@intel.com>

```c
From patchwork Sun Mar 22 12:35:57 2020
Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
X-Patchwork-Submitter: Liu Yi L <yi.l.liu@intel.com>
X-Patchwork-Id: 11451731
Return-Path: <SRS0=VAWz=5H=vger.kernel.org=kvm-owner@kernel.org>
Received: from mail.kernel.org (pdx-korg-mail-1.web.codeaurora.org
 [172.30.200.123])
	by pdx-korg-patchwork-2.web.codeaurora.org (Postfix) with ESMTP id DFB2F1731
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 22 Mar 2020 12:31:07 +0000 (UTC)
Received: from vger.kernel.org (vger.kernel.org [209.132.180.67])
	by mail.kernel.org (Postfix) with ESMTP id BED2820732
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 22 Mar 2020 12:31:07 +0000 (UTC)
Received: (majordomo@vger.kernel.org) by vger.kernel.org via listexpand
        id S1727241AbgCVMbG (ORCPT
        <rfc822;patchwork-kvm@patchwork.kernel.org>);
        Sun, 22 Mar 2020 08:31:06 -0400
Received: from mga17.intel.com ([192.55.52.151]:58572 "EHLO mga17.intel.com"
        rhost-flags-OK-OK-OK-OK) by vger.kernel.org with ESMTP
        id S1726990AbgCVMaj (ORCPT <rfc822;kvm@vger.kernel.org>);
        Sun, 22 Mar 2020 08:30:39 -0400
IronPort-SDR: 
 24R/LK/4pB9as9K3EP/wnFu/dWhdtSX3AJMagDKglXfXHHTMuq38ryD8G23hJgzpypTyyoIOOR
 JAQ7Tc9rcWtQ==
X-Amp-Result: SKIPPED(no attachment in message)
X-Amp-File-Uploaded: False
Received: from orsmga008.jf.intel.com ([10.7.209.65])
  by fmsmga107.fm.intel.com with ESMTP/TLS/ECDHE-RSA-AES256-GCM-SHA384;
 22 Mar 2020 05:30:38 -0700
IronPort-SDR: 
 uSfxUwmnjJc1rtK+WmXQLMJcsd8CSy8SPd1SvU5zGoXlsFBUNjtT+ToMX5uo2k7Tmehe46lfs/
 vY8cMjO59o5Q==
X-ExtLoop1: 1
X-IronPort-AV: E=Sophos;i="5.72,292,1580803200";
   d="scan'208";a="239664350"
Received: from jacob-builder.jf.intel.com ([10.7.199.155])
  by orsmga008.jf.intel.com with ESMTP; 22 Mar 2020 05:30:37 -0700
From: Liu Yi L <yi.l.liu@intel.com>
To: qemu-devel@nongnu.org, alex.williamson@redhat.com,
        peterx@redhat.com
Cc: eric.auger@redhat.com, pbonzini@redhat.com, mst@redhat.com,
        david@gibson.dropbear.id.au, kevin.tian@intel.com,
        yi.l.liu@intel.com, jun.j.tian@intel.com, yi.y.sun@intel.com,
        kvm@vger.kernel.org, hao.wu@intel.com, jean-philippe@linaro.org
Subject: [PATCH v1 00/22] intel_iommu: expose Shared Virtual Addressing to VMs
Date: Sun, 22 Mar 2020 05:35:57 -0700
Message-Id: <1584880579-12178-1-git-send-email-yi.l.liu@intel.com>
X-Mailer: git-send-email 2.7.4
Sender: kvm-owner@vger.kernel.org
Precedence: bulk
List-ID: <kvm.vger.kernel.org>
X-Mailing-List: kvm@vger.kernel.org

Shared Virtual Addressing (SVA), a.k.a, Shared Virtual Memory (SVM) on
Intel platforms allows address space sharing between device DMA and
applications. SVA can reduce programming complexity and enhance security.

This QEMU series is intended to expose SVA usage to VMs. i.e. Sharing
guest application address space with passthru devices. This is called
vSVA in this series. The whole vSVA enabling requires QEMU/VFIO/IOMMU
changes.

The high-level architecture for SVA virtualization is as below, the key
design of vSVA support is to utilize the dual-stage IOMMU translation (
also known as IOMMU nesting translation) capability in host IOMMU.

    .-------------.  .---------------------------.
    |   vIOMMU    |  | Guest process CR3, FL only|
    |             |  '---------------------------'
    .----------------/
    | PASID Entry |--- PASID cache flush -
    '-------------'                       |
    |             |                       V
    |             |                CR3 in GPA
    '-------------'
Guest
------| Shadow |--------------------------|--------
      v        v                          v
Host
    .-------------.  .----------------------.
    |   pIOMMU    |  | Bind FL for GVA-GPA  |
    |             |  '----------------------'
    .----------------/  |
    | PASID Entry |     V (Nested xlate)
    '----------------\.------------------------------.
    |             |   |SL for GPA-HPA, default domain|
    |             |   '------------------------------'
    '-------------'
Where:
 - FL = First level/stage one page tables
 - SL = Second level/stage two page tables

The complete vSVA kernel upstream patches are divided into three phases:
    1. Common APIs and PCI device direct assignment
    2. IOMMU-backed Mediated Device assignment
    3. Page Request Services (PRS) support

This QEMU patchset is aiming for the phase 1 and phase 2. It is based
on the two kernel series below.
[1] [PATCH V10 00/11] Nested Shared Virtual Address (SVA) VT-d support:
https://lkml.org/lkml/2020/3/20/1172
[2] [PATCH v1 0/8] vfio: expose virtual Shared Virtual Addressing to VMs
https://lkml.org/lkml/2020/3/22/116

There are roughly two parts:
 1. Introduce HostIOMMUContext as abstract of host IOMMU. It provides explicit
    method for vIOMMU emulators to communicate with host IOMMU. e.g. propagate
    guest page table binding to host IOMMU to setup dual-stage DMA translation
    in host IOMMU and flush iommu iotlb.
 2. Setup dual-stage IOMMU translation for Intel vIOMMU. Includes 
    - Check IOMMU uAPI version compatibility and VFIO Nesting capabilities which
      includes hardware compatibility (stage 1 format) and VFIO_PASID_REQ
      availability. This is preparation for setting up dual-stage DMA translation
      in host IOMMU.
    - Propagate guest PASID allocation and free request to host.
    - Propagate guest page table binding to host to setup dual-stage IOMMU DMA
      translation in host IOMMU.
    - Propagate guest IOMMU cache invalidation to host to ensure iotlb
      correctness.

The complete QEMU set can be found in below link:
https://github.com/luxis1999/qemu.git: sva_vtd_v10_v1

Complete kernel can be found in:
https://github.com/luxis1999/linux-vsva.git: vsva-linux-5.6-rc6

Tests: basci vSVA functionality test, VM reboot/shutdown/crash, kernel build in
guest, boot VM with vSVA disabled, full comapilation.

Regards,
Yi Liu

Changelog:
	- RFC v3.1 -> Patch v1:
	  a) Implement HostIOMMUContext in QOM manner.
	  b) Add pci_set/unset_iommu_context() to register HostIOMMUContext to
	     vIOMMU, thus the lifecircle of HostIOMMUContext is awared in vIOMMU
	     side. In such way, vIOMMU could use the methods provided by the
	     HostIOMMUContext safely.
	  c) Add back patch "[RFC v3 01/25] hw/pci: modify pci_setup_iommu() to set PCIIOMMUOps"
	  RFCv3.1: https://patchwork.kernel.org/cover/11397879/

	- RFC v3 -> v3.1:
	  a) Drop IOMMUContext, and rename DualStageIOMMUObject to HostIOMMUContext.
	     HostIOMMUContext is per-vfio-container, it is exposed to  vIOMMU via PCI
	     layer. VFIO registers a PCIHostIOMMUFunc callback to PCI layer, vIOMMU
	     could get HostIOMMUContext instance via it.
	  b) Check IOMMU uAPI version by VFIO_CHECK_EXTENSION
	  c) Add a check on VFIO_PASID_REQ availability via VFIO_GET_IOMMU_IHNFO
	  d) Reorder the series, put vSVA linux header file update in the beginning
	     put the x-scalable-mode option mofification in the end of the series.
	  e) Dropped patch "[RFC v3 01/25] hw/pci: modify pci_setup_iommu() to set PCIIOMMUOps"
	  RFCv3: https://patchwork.kernel.org/cover/11356033/

	- RFC v2 -> v3:
	  a) Introduce DualStageIOMMUObject to abstract the host IOMMU programming
	  capability. e.g. request PASID from host, setup IOMMU nesting translation
	  on host IOMMU. The pasid_alloc/bind_guest_page_table/iommu_cache_flush
	  operations are moved to be DualStageIOMMUOps. Thus, DualStageIOMMUObject
	  is an abstract layer which provides QEMU vIOMMU emulators with an explicit
	  method to program host IOMMU.
	  b) Compared with RFC v2, the IOMMUContext has also been updated. It is
	  modified to provide an abstract for vIOMMU emulators. It provides the
	  method for pass-through modules (like VFIO) to communicate with host IOMMU.
	  e.g. tell vIOMMU emulators about the IOMMU nesting capability on host side
	  and report the host IOMMU DMA translation faults to vIOMMU emulators.
	  RFC v2: https://www.spinics.net/lists/kvm/msg198556.html

	- RFC v1 -> v2:
	  Introduce IOMMUContext to abstract the connection between VFIO
	  and vIOMMU emulators, which is a replacement of the PCIPASIDOps
	  in RFC v1. Modify x-scalable-mode to be string option instead of
	  adding a new option as RFC v1 did. Refined the pasid cache management
	  and addressed the TODOs mentioned in RFC v1. 
	  RFC v1: https://patchwork.kernel.org/cover/11033657/


Eric Auger (1):
  scripts/update-linux-headers: Import iommu.h

Liu Yi L (21):
  header file update VFIO/IOMMU vSVA APIs
  vfio: check VFIO_TYPE1_NESTING_IOMMU support
  hw/iommu: introduce HostIOMMUContext
  hw/pci: modify pci_setup_iommu() to set PCIIOMMUOps
  hw/pci: introduce pci_device_set/unset_iommu_context()
  intel_iommu: add set/unset_iommu_context callback
  vfio: init HostIOMMUContext per-container
  vfio/common: check PASID alloc/free availability
  intel_iommu: add virtual command capability support
  intel_iommu: process PASID cache invalidation
  intel_iommu: add PASID cache management infrastructure
  vfio: add bind stage-1 page table support
  intel_iommu: bind/unbind guest page table to host
  intel_iommu: replay guest pasid bindings to host
  intel_iommu: replay pasid binds after context cache invalidation
  intel_iommu: do not pass down pasid bind for PASID #0
  vfio: add support for flush iommu stage-1 cache
  intel_iommu: process PASID-based iotlb invalidation
  intel_iommu: propagate PASID-based iotlb invalidation to host
  intel_iommu: process PASID-based Device-TLB invalidation
  intel_iommu: modify x-scalable-mode to be string option

 hw/Makefile.objs                      |    1 +
 hw/alpha/typhoon.c                    |    6 +-
 hw/arm/smmu-common.c                  |    6 +-
 hw/hppa/dino.c                        |    6 +-
 hw/i386/amd_iommu.c                   |    6 +-
 hw/i386/intel_iommu.c                 | 1221 ++++++++++++++++++++++++++++++++-
 hw/i386/intel_iommu_internal.h        |  118 ++++
 hw/i386/trace-events                  |    6 +
 hw/iommu/Makefile.objs                |    1 +
 hw/iommu/host_iommu_context.c         |  178 +++++
 hw/pci-host/designware.c              |    6 +-
 hw/pci-host/pnv_phb3.c                |    6 +-
 hw/pci-host/pnv_phb4.c                |    6 +-
 hw/pci-host/ppce500.c                 |    6 +-
 hw/pci-host/prep.c                    |    6 +-
 hw/pci-host/sabre.c                   |    6 +-
 hw/pci/pci.c                          |   53 +-
 hw/ppc/ppc440_pcix.c                  |    6 +-
 hw/ppc/spapr_pci.c                    |    6 +-
 hw/s390x/s390-pci-bus.c               |    8 +-
 hw/vfio/common.c                      |  257 ++++++-
 hw/vfio/pci.c                         |   13 +
 hw/virtio/virtio-iommu.c              |    6 +-
 include/hw/i386/intel_iommu.h         |   62 +-
 include/hw/iommu/host_iommu_context.h |  116 ++++
 include/hw/pci/pci.h                  |   18 +-
 include/hw/pci/pci_bus.h              |    2 +-
 include/hw/vfio/vfio-common.h         |    4 +
 linux-headers/linux/iommu.h           |  378 ++++++++++
 linux-headers/linux/vfio.h            |  127 ++++
 scripts/update-linux-headers.sh       |    2 +-
 31 files changed, 2599 insertions(+), 44 deletions(-)
 create mode 100644 hw/iommu/Makefile.objs
 create mode 100644 hw/iommu/host_iommu_context.c
 create mode 100644 include/hw/iommu/host_iommu_context.h
 create mode 100644 linux-headers/linux/iommu.h
```
