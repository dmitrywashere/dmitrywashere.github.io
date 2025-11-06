---
layout: post
title: "The Storage Refresh is Dead (And Your Downtime Just Got the Memo)"
date: 2025-11-06
categories: [se, purestorage, flasharray, evergreen, ndu, homelab, innovation]
published: true 
---

# 💡 Evergreen: The Storage Refresh is Dead (And Your Downtime Just Got the Memo)

I know, I know. Another post about non-disruptive upgrades. Trust me, I was ready to skip this topic until I realized I had to actually do one three days ago. Let's ditch the marketing slides and look at what this looks like when a human (me) is holding the cables.

<img src="/assets/images/ndu/0peace.jpeg" alt="Peace" width="650">

Let’s be honest. Planning a storage refresh is the worst. It’s a project that starts with technical debt and ends with budget pain. It means months of planning, coordinating large-scale downtime windows, managing complex data migrations that often introduce risk, retraining staff on a fundamentally new platform, and that sinking feeling when you look at the surprise renewal invoice years down the line.

At **Pure Storage**, we fundamentally rejected that cycle. We realized that for storage to be truly modern, it must avoid disruption entirely. Our answer isn't a lease program or a short-term marketing slogan; it's an architectural commitment called **Evergreen**.

Evergreen is not a promotion; it’s a design philosophy that ensures your array is **always upgradable and always non-disruptive**. This design philosophy, known formally as **Evergreen//Forever**, means you buy the storage platform once, and it continuously improves underneath you—controllers, media, and software—while your mission-critical workloads remain online. No more forklift refreshes. No more data migrations. No "do-overs" every few years.

With over **30,000+ non-disruptive controller upgrades** successfully completed across our customer base, we've proven that "non-disruptive everything" isn't aspirational marketing; it’s operational reality at scale.

---

## The Core Problem: Why Storage Gets Old (and Expensive)

Traditional storage vendors built their platforms with legacy disk in mind, and the core components—the controllers and the data—were tightly coupled. This means that when the controllers reach their end-of-life, the only way to upgrade is to perform a full system replacement and migrate all your data. This model is engineered to be expensive and disruptive.

We designed our system for the age of flash from the beginning, with a clean, modular architecture that allows us to separate the two critical elements:

### 1. Architectural Commitment: The Stateless Controller

The fundamental engine of Evergreen is the **stateless controller**. All configuration information and, most importantly, **all persistent data lives within the array's data layer, independent of the controller heads**.

This is the key design choice that makes continual upgrades routine. Because the controllers are stateless, they can be upgraded across generations—like the move from X20R2 to X20R4—without touching your host data paths, host connectivity, or the volumes themselves. The data simply moves to the new, more powerful brain.

### 2. Operational Commitment: Non-Disruptive Everything (NDU)

The platform is built to upgrade and maintain in place. This is delivered through:

* **Active/Active Front-End Paths:** Host multipathing always preserves I/O throughout the upgrade process.
* **Rolling Orchestration:** Controllers and software are updated in a rolling fashion, ensuring that one controller is always available to manage the entire workload.

This means cross-generation controller upgrades and major software releases (Purity Operating Environment) are standard operating procedure and can even be executed **self-service from Pure1** when it suits your maintenance windows.

---

## Media That Lasts: The DirectFlash Advantage

Longevity isn't just about the controllers; it's about the flash media itself. Instead of hiding high-performance NAND behind generic commodity SSD controllers, we manage the flash directly in software via **DirectFlash technology**.

This approach gives us complete, system-wide control, leading to superior endurance and predictable performance:

* **Reduced Write Amplification (WA):** By managing placement and scheduling directly, we lower write amplification by **more than 3x versus commodity SSDs**. This directly extends flash endurance.
* **Consistent Latency:** System-level management works around NAND Program/Erase (P/E) cycles, ensuring you get consistent, predictable latency even as we scale to higher-density QLC media.

For added protection, the **Evergreen Forever Component Replacement** ensures that while your subscription is active, media wear is never a concern. You receive like-for-like or better replacements to keep your array performing to its original specification.

---

## 🛠️ From X20R2 to X20R4: The Evergreen NDU, Step-by-Step Validation

The best way to prove this architectural philosophy is to see it in action. I'm currently executing a leap-frog upgrade in my lab, moving my array from a **FlashArray//X20R2 to a newer FlashArray//X20R4**—a move across major generations of compute. The entire controller swap takes about two to four hours and happens with **zero interruption to host I/O**.

### Phase 2: Controller 0 (CT0) Swap (Mixed-Mode Operation)

The upgrade is performed one controller at a time, allowing the array to run in an active/active, mixed-mode configuration during the transition.

| Step | Action | Key Command/Note |
| :--- | :--- | :--- |
| **1.** | **Isolate I/O from CT0.** | Failover to secondary mode: `purewes controller setattr cto --mode secondary`. |
| **2.** | **Set I/O Preference (Guaranteed Path).** | On the active partner (**CT1**), explicitly direct all data paths: `puredb prefer CT1`. |
| **3.** | **Gracefully Stop CT0.** | Halt the Purity Operating Environment: `pureadm stop`. |
| **4.** | **Remove Old CT0.** | Disconnect all data and management cables (but **not power cords**), remove the old **FlashArray//X CT0**. |
| **5.** | **Install New CT0.** | Install new **FlashArray//XR4 CT0**. Move supported PCIe cards (HBAs, accelerators) from old controller. |
| **6.** | **Initialize & Recable.** | Run `puresetup replace` twice on the new CT0 console to apply tunables, reboot, and display exact port re-cabling instructions. Reconnect cables as instructed. |

Replace the Power Supplies (Optional)

Here is the GUI image of the array I am upgrading after I pulled the old power supply out.

<img src="/assets/images/ndu/1Before.png" alt="Before" width="650">

Here is what it looks like from the back:

<img src="/assets/images/ndu/2ReplacePowerSupply.png" alt="Power Supply" width="650">


### Phase 3: Controller 1 (CT1) Swap (Full Upgrade)

With the new CT0 now stabilized and serving I/O, we repeat the procedure to replace the final legacy component:

| Step | Action | Key Command/Note |
| :--- | :--- | :--- |
| **7.** | **Isolate I/O from CT1.** | Failover to secondary mode: `purewes controller setattr ct1 --mode secondary`. |
| **8.** | **Set I/O Preference (Guaranteed Path).** | On the newly upgraded **CT0**, set preference: `puredb prefer CT0`. |
| **9.** | **Gracefully Stop CT1.** | Halt the Purity Operating Environment: `pureadm stop`. |
| **10.**| **Remove Old CT1.** | Disconnect all data/management cables, remove the old **FlashArray//X CT1**. |
| **11.**| **Install New CT1.** | Install new **FlashArray//XR4 CT1**. Move remaining supported PCIe cards. |
| **12.**| **Initialize & Recable.** | Run `puresetup replace` twice on the new CT1 console. Reconnect all remaining cables. |

<img src="/assets/images/ndu/3CT0removed.png" alt="CT0 removed" width="650">

CT0 was removed and the end user traffic is flowing uniterrupted. It's like magic. 

You can see the difference in controller HW between CT0 and CT1 - this is a cross genegational upgrade, remember?

<img src="/assets/images/ndu/4NewXR4CTO.png" alt="New CT0" width="650">

Here is the CLI output:

<img src="/assets/images/ndu/5CLICheck.png" alt="CLI" width="650">

Still no traffic interruption:

<img src="/assets/images/ndu/7NoTrafficInterruption.png" alt="NDU" width="650">

### Phase 4: Finalization

All that is left to do is to do the last validation and update the time zone.

<img src="/assets/images/ndu/9FinalStep.png" alt="Time Zone" width="650">

The array is now fully on the new **FlashArray//XR4** controllers!

<img src="/assets/images/ndu/8NDUDone.png" alt="Power Supply" width="650">

* **Restore I/O Balance:** Remove the preference setting to allow full active/active I/O to both controllers: `puredb prefer ""`.
* **Final Verification:** Run final health checks, verify status, and test failover to confirm full redundancy.
* **Final Tidy-up:** Remove maintenance alert tags, apply the new model label, and pack the old hardware for return.

The core principle remains true: **your data stays in place, I/O is continuously served, and the disruption is limited entirely to the hardware swap, not your applications.**

---

## The Promise of Predictability: Flat & Fair

The Evergreen architectural promise is paired with a financial one: **Flat & Fair** renewal pricing.

This commitment means maintenance renewal costs remain consistent and predictable over time (with clear, documented exceptions, like severe inflation events), allowing IT teams to plan OPEX budgets without fear of "maintenance extortion".

Furthermore, every innovation we deliver in the Purity Operating Environment—all features, all advancements—is **included with the Evergreen//Forever subscription**. There are no surprise license tier upsells or new feature lines to budget for.

---

## What This Means For Your Team

If you’re done planning around disruptive refresh cycles, retraining teams on new platforms, or budgeting for "gotcha" renewals, Evergreen was built to end that. You gain:

* **Investment Protection:** You buy the data capacity once and maintain its use forever. You can expand capacity or upgrade controllers with trade-in credits, without rebuying the terabytes you already own.
* **Operational Continuity:** Your applications stay online, your teams focus on innovation (not migration), and your array continuously gets better with age.
* **Budget Clarity:** Predictable cost growth allows for confident, long-term OPEX planning.

Evergreen marries a flash-native architecture, stateless upgrade design, and inclusive subscription model to deliver non-disruptive operations, predictable maintenance costs, and longer-lived flash—by design, not by exception.

---

**Ready to say goodbye to the storage refresh cycle?**