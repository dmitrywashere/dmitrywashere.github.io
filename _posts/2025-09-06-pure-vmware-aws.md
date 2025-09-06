---
layout: post
title: "AWS Elastic VMware Service with Pure Storage Cloud Block Store"
date: 2025-09-06
categories: [data, cloud, aws, evs, cbs, purestorage]
published: true
---

I spent an insightful hour this week at the "Clear the Path to VMware in AWS" webinar, and I wanted to share my key takeaways for fellow IT professionals trying to understand the intricacies of this implementation. The session, featuring Erin Stevens and Kyle Grossmiller from Pure Storage, alongside Ben Lipman from Amazon Web Services, cut through the marketing fluff and got right to the architectural heart of the matter.


<img src="/assets/images/CBSonEVS/Pure VMware AWS.png" alt="Illustration of AWS EVS" width="650">

## The Hybrid Cloud Problem Isn't Going Away

The session kicked off by framing a reality we all live in: the hybrid cloud is not a temporary phase. The challenge is managing data that needs to exist and perform across on-prem data centers and various cloud environments.

A major hurdle, as the speakers noted, is that cloud migrations often stall because mission-critical applications aren't built to run as native AWS services. Think legacy applications on older OS versions that can't be refactored overnight. This is where the core of the discussion began.

## Amazon EVS: The "Easy Button" for VMware in the Cloud

Ben Lipman from AWS provided a fantastic overview of **Amazon Elastic VMware Service (EVS)**. The key takeaway here is its simplicity and familiarity: **EVS is a fully managed, native VMware SDDC running on dedicated bare-metal hosts within an AWS region.**

What this means for IT pros:
* **It's the Real Deal:** You get the full VMware experience you already know—vCenter, SDDC Manager, NSX Manager—without any modifications. Your existing scripts and operational knowledge apply directly.
* **Bare Metal as a Service:** AWS handles the heavy lifting of deploying and configuring the underlying SDDC. Ben noted a four-host cluster can be provisioned in about four hours, with new hosts added in around 30 minutes. This is a massive operational win.
* **Full Admin Control:** This was a critical point. Unlike some managed services, EVS provides full root-level administrative access. You can SSH into appliances, check logs, and install third-party plugins just as you would on-premises, giving you the granular control necessary for enterprise environments.

## Pure Storage Cloud Block Store: The Enterprise-Grade Storage Layer

This is where Kyle Grossmiller from Pure Storage explained how their solution complements EVS. **Pure Storage Cloud Block Store (CBS)** is essentially the Purity operating environment from their on-premises FlashArray, running natively as software in AWS.

For more information, see the official blog post: [Pure Storage Cloud Service for Amazon Elastic VMware Service | Pure Storage Blog](https://blog.purestorage.com/products/pure-storage-cloud-service-for-amazon-elastic-vmware-service/)

This creates a seamless storage experience with powerful benefits:
* **Consistent Enterprise Features:** You get all the features you rely on in the data center—industry-leading data reduction, thin provisioning, SafeMode for ransomware protection, and asynchronous replication—extended directly into your AWS environment.
* **Massive DR Cost Savings:** This was one of the most compelling points. Because Pure's replication sends only unique, reduced data, egress costs for disaster recovery are slashed. If your 100TB on-prem array has a 4:1 data reduction, you only replicate and pay egress for 25TB of data to AWS. For failback, you only send the changed data back. This is a huge TCO advantage.
* **Pay-as-You-Grow Efficiency:** CBS deploys with a minimal footprint and can be scaled non-disruptively. This aligns perfectly with the cloud consumption model and avoids overprovisioning.

## NVMe/TCP: The Performance Linchpin

Both Kyle and Ben heavily emphasized using **NVMe/TCP** as the storage protocol between EVS and Cloud Block Store. While iSCSI is supported, NVMe/TCP is the modern, more efficient choice. It delivers lower latency, higher performance (up to 2.5x in some tests), and is more CPU-efficient, which means your expensive ESXi hosts spend their cycles running your VMs, not managing storage I/O.

## Actionable Lessons Learned (The Pro-Tips)

The session concluded with practical advice from the preview phase, which is invaluable for anyone planning a deployment:
* **Co-locate everything:** Deploy EVS and CBS in the same region, VPC, and ideally, the same Availability Zone to minimize latency and data traversal costs.
* **Match your MTU sizes:** Ensure the MTU size is consistent between EVS and CBS (8500 is recommended).
* **Use the right VLANs:** EVS has dedicated expansion VLANs for external storage. Use these to leverage the dual 75Gb network interfaces for high availability.
* **Default to NVMe/TCP:** It's the clear winner for performance and efficiency.

## Detailed Implementation Guide

For detailed deployment instructions, please refer to the official guide: 
[Elastic VMware Service and Pure Cloud Block Store iSCSI Implementation Guide](https://www.purestorage.com/docs/assets/pdf/iscsi-implementation-guide-evs-cbs.pdf)

## My Verdict

The Pure Storage + Amazon EVS partnership offers a robust, familiar, and surprisingly cost-effective solution for extending a VMware footprint into the AWS cloud. It removes major migration roadblocks for legacy apps, provides the enterprise-grade storage features that AWS's native block storage lacks, and gives IT teams the full control they are used to. It's a powerful combination that genuinely bridges the gap between on-prem and cloud.