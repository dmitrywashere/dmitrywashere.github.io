---
layout: post
title: "Evergreen: Killing the Storage Refresh Cycle—Forever"
date: 2025-11-06
categories: [se, purestorage, flasharray, evergreen, ndu, homelab]
published: true 
---

# 💡 Evergreen: Killing the Storage Refresh Cycle—Forever

Let’s be honest. Planning a storage refresh is the worst. It’s a project that starts with technical debt and ends with budget pain. It means months of planning, coordinating large-scale downtime windows, managing complex data migrations that often introduce risk, retraining staff on a fundamentally new platform, and that sinking feeling when you look at the surprise renewal invoice years down the line.

At **Pure Storage**, we fundamentally rejected that cycle. We realized that for storage to be truly modern, it must avoid disruption entirely. Our answer isn't a lease program or a short-term marketing slogan; it's an architectural commitment called **Evergreen**.

[cite_start]Evergreen is not a promotion; it’s a design philosophy that ensures your array is **always upgradable and always non-disruptive**[cite: 1]. This design philosophy, known formally as **Evergreen//Forever**, means you buy the storage platform once, and it continuously improves underneath you—controllers, media, and software—while your mission-critical workloads remain online. No more forklift refreshes. No more data migrations. [cite_start]No "do-overs" every few years[cite: 1].

[cite_start]With over **30,000+ non-disruptive controller upgrades** successfully completed across our customer base, we've proven that "non-disruptive everything" isn't aspirational marketing; it’s operational reality at scale[cite: 1].

---

## The Core Problem: Why Storage Gets Old (and Expensive)

Traditional storage vendors built their platforms with legacy disk in mind, and the core components—the controllers and the data—were tightly coupled. This means that when the controllers reach their end-of-life, the only way to upgrade is to perform a full system replacement and migrate all your data. This model is engineered to be expensive and disruptive.

We designed our system for the age of flash from the beginning, with a clean, modular architecture that allows us to separate the two critical elements:

### 1. Architectural Commitment: The Stateless Controller

[cite_start]The fundamental engine of Evergreen is the **stateless controller**[cite: 1]. [cite_start]All configuration information and, most importantly, **all persistent data lives within the array's data layer, independent of the controller heads**[cite: 1].

This is the key design choice that makes continual upgrades routine. Because the controllers are stateless, they can be upgraded across generations—like the move from X20R2 to X20R4—without touching your host data paths, host connectivity, or the volumes themselves. The data simply moves to the new, more powerful brain.

### 2. Operational Commitment: Non-Disruptive Everything (NDU)

[cite_start]The platform is built to upgrade and maintain in place[cite: 1]. This is delivered through:

* [cite_start]**Active/Active Front-End Paths:** Host multipathing always preserves I/O throughout the upgrade process[cite: 1].
* [cite_start]**Rolling Orchestration:** Controllers and software are updated in a rolling fashion, ensuring that one controller is always available to manage the entire workload[cite: 1].

[cite_start]This means cross-generation controller upgrades and major software releases (Purity Operating Environment) are standard operating procedure and can even be executed **self-service from Pure1** when it suits your maintenance windows[cite: 1].

---

## Media That Lasts: The DirectFlash Advantage

Longevity isn't just about the controllers; it's about the flash media itself. [cite_start]Instead of hiding high-performance NAND behind generic commodity SSD controllers, we manage the flash directly in software via **DirectFlash technology**[cite: 1].

This approach gives us complete, system-wide control, leading to superior endurance and predictable performance:

* [cite_start]**Reduced Write Amplification (WA):** By managing placement and scheduling directly, we lower write amplification by **more than 3x versus commodity SSDs**[cite: 1]. This directly extends flash endurance.
* [cite_start]**Consistent Latency:** System-level management works around NAND Program/Erase (P/E) cycles, ensuring you get consistent, predictable latency even as we scale to higher-density QLC media[cite: 1].

For added protection, the **Evergreen Forever Component Replacement** ensures that while your subscription is active, media wear is never a concern. [cite_start]You receive like-for-like or better replacements to keep your array performing to its original specification[cite: 1].

---

## 🛠️ From X20R2 to X20R4: The Evergreen NDU, Step-by-Step Validation

The best way to prove this architectural philosophy is to see it in action. I'm currently executing a leap-frog upgrade in my lab, moving my array from a **FlashArray//X20R2 to a newer FlashArray//X20R4**—a move across major generations of compute. [cite_start]The entire controller swap takes about two to four hours and happens with **zero interruption to host I/O**[cite: 136, 576].

***(Note: I will be adding screen captures of this process here!)***

### Phase 1: Preparation

[cite_start]Before the physical work, we ensure the array is ready for a hardware swap while staying fully active on the remaining controller[cite: 160]:

| Task | Detail | Requirement |
| :--- | :--- | :--- |
| **Purity Check** | [cite_start]Array must be running a minimum of **Purity 6.4.8 or later**[cite: 78]. | Essential for compatibility. |
| **Pre-Check & Logs** | [cite_start]Run health checks, resolve alerts, and send logs to Pure Support[cite: 161, 162, 169]. | Ensures stability during the process. |
| **Power Input** | [cite_start]Verify the rack has **high-line power (200 VAC or higher)**, as required by the new R4 controllers[cite: 151]. | Critical hardware requirement. |

### Phase 2: Controller 0 (CT0) Swap (Mixed-Mode Operation)

[cite_start]The upgrade is performed one controller at a time, allowing the array to run in an active/active, mixed-mode configuration during the transition[cite: 576].

| Step | Action | Key Command/Note |
| :--- | :--- | :--- |
| **1.** | **Isolate I/O from CT0.** | [cite_start]Failover to secondary mode: `purewes controller setattr cto --mode secondary`[cite: 702]. |
| **2.** | **Set I/O Preference (Guaranteed Path).** | [cite_start]On the active partner (**CT1**), explicitly direct all data paths: `puredb prefer CT1`[cite: 738]. |
| **3.** | **Gracefully Stop CT0.** | [cite_start]Halt the Purity Operating Environment: `pureadm stop`[cite: 739]. |
| **4.** | **Remove Old CT0.** | [cite_start]Disconnect all data and management cables (but **not power cords**) [cite: 747, 748, 753][cite_start], remove the old **FlashArray//X CT0**[cite: 759]. |
| **5.** | **Install New CT0.** | Install new **FlashArray//XR4 CT0**. [cite_start]Move supported PCIe cards (HBAs, accelerators) from old controller[cite: 779, 780]. |
| **6.** | **Initialize & Recable.** | [cite_start]Run `puresetup replace` twice on the new CT0 console to apply tunables, reboot, and display exact port re-cabling instructions[cite: 1341, 1342]. Reconnect cables as instructed. |

### Phase 3: Controller 1 (CT1) Swap (Full Upgrade)

With the new CT0 now stabilized and serving I/O, we repeat the procedure to replace the final legacy component:

| Step | Action | Key Command/Note |
| :--- | :--- | :--- |
| **7.** | **Isolate I/O from CT1.** | [cite_start]Failover to secondary mode: `purewes controller setattr ct1 --mode secondary`[cite: 1452]. |
| **8.** | **Set I/O Preference (Guaranteed Path).** | [cite_start]On the newly upgraded **CT0**, set preference: `puredb prefer CT0`[cite: 1484]. |
| **9.** | **Gracefully Stop CT1.** | [cite_start]Halt the Purity Operating Environment: `pureadm stop`[cite: 1485]. |
| **10.**| **Remove Old CT1.** | [cite_start]Disconnect all data/management cables, remove the old **FlashArray//X CT1**[cite: 1491, 1505]. |
| **11.**| **Install New CT1.** | Install new **FlashArray//XR4 CT1**. [cite_start]Move remaining supported PCIe cards[cite: 1523]. |
| **12.**| **Initialize & Recable.** | Run `puresetup replace` twice on the new CT1 console. [cite_start]Reconnect all remaining cables[cite: 2076]. |

### Phase 4: Finalization

The array is now fully on the new **FlashArray//XR4** controllers!

* [cite_start]**Restore I/O Balance:** Remove the preference setting to allow full active/active I/O to both controllers: `puredb prefer ""`[cite: 2160].
* [cite_start]**Final Verification:** Run final health checks, verify status, and test failover to confirm full redundancy[cite: 2155, 2161].
* [cite_start]**Final Tidy-up:** Remove maintenance alert tags, apply the new model label, and pack the old hardware for return[cite: 2275, 2246, 2282].

The core principle remains true: **your data stays in place, I/O is continuously served, and the disruption is limited entirely to the hardware swap, not your applications.**

---

## The Promise of Predictability: Flat & Fair

[cite_start]The Evergreen architectural promise is paired with a financial one: **Flat & Fair** renewal pricing[cite: 1].

[cite_start]This commitment means maintenance renewal costs remain consistent and predictable over time (with clear, documented exceptions, like severe inflation events), allowing IT teams to plan OPEX budgets without fear of "maintenance extortion"[cite: 1].

[cite_start]Furthermore, every innovation we deliver in the Purity Operating Environment—all features, all advancements—is **included with the Evergreen//Forever subscription**[cite: 1]. There are no surprise license tier upsells or new feature lines to budget for.

---

## What This Means For Your Team

If you’re done planning around disruptive refresh cycles, retraining teams on new platforms, or budgeting for "gotcha" renewals, Evergreen was built to end that. You gain:

* **Investment Protection:** You buy the data capacity once and maintain its use forever. [cite_start]You can expand capacity or upgrade controllers with trade-in credits, without rebuying the terabytes you already own[cite: 1].
* **Operational Continuity:** Your applications stay online, your teams focus on innovation (not migration), and your array continuously gets better with age.
* **Budget Clarity:** Predictable cost growth allows for confident, long-term OPEX planning.

[cite_start]Evergreen marries a flash-native architecture, stateless upgrade design, and inclusive subscription model to deliver non-disruptive operations, predictable maintenance costs, and longer-lived flash—by design, not by exception[cite: 1].

---

**Ready to say goodbye to the storage refresh cycle?**
