---
title: Terminology
status: working
last_reviewed: 2026-03-07
owner: Rognheim
---

# :material-text-box:


## **Terminology**

---

### **Introduction**

Self-hosting terminology encompasses the key concepts and vocabulary used to describe the process of hosting and managing applications, services, and data on one's own servers or devices, rather than relying on third-party providers. Understanding self-hosting terminology is essential for effectively setting up, maintaining, and troubleshooting self-hosted environments. This includes concepts such as server hardware and software, virtualization, networking, security, and storage management. Familiarity with these terms enables users to make informed decisions and optimize their self-hosting experience.

---

### **Basic concepts**



---


### **Networking**

#### OSI Model

The OSI (Open Systems Interconnection) Model is a conceptual framework that standardizes the functions of a communication system into seven distinct categories, or layers. These layers, from highest to lowest, are: Application, Presentation, Session, Transport, Network, Data Link, and Physical. Each layer performs specific functions and communicates with the layers directly above and below it. Understanding the OSI Model can help troubleshoot network issues, design effective network architectures, and implement appropriate security measures.

#### Uplink

In networking, an "uplink" usually refers to a connection that moves data from a local or smaller network to a larger network. In the context of a network switch, the uplink port is used to connect the switch to a broader network, such as a router, a server, or another switch, effectively expanding the network's reach.

The uplink port is designed to avoid conflicts or packet collisions that could occur when two devices try to send data at the same time. In a typical setup, a network switch might have multiple ports for connecting devices within a local network, such as computers, printers, and other peripherals, and one or more uplink ports for connecting to a larger network or the internet.

Keep in mind that the term "uplink" can be used differently depending on the context. In satellite communications, for example, an uplink refers to a radio signal that is sent from Earth to a spacecraft or satellite.

---

#### Attenuation

Attenuation in the context of networking refers to the loss of signal strength that occurs as data travels over the length of a network cable or over a wireless connection.

In wired networks, attenuation can be caused by a variety of factors, including the length of the cable, the quality of the cable, and electromagnetic interference from other sources. Signal loss increases with the distance the signal must travel, so there are limits on the length of network cables to ensure that data can be transmitted effectively.

In wireless networks, attenuation can be caused by distance, physical obstacles (like walls and buildings), and interference from other wireless signals. This is why the signal strength of Wi-Fi networks decreases as you move further away from the router, or if there are walls or other obstacles between your device and the router.

Attenuation is usually measured in decibels (dB). The higher the dB value, the greater the attenuation, and therefore the weaker the signal. To counteract attenuation, repeaters or signal amplifiers may be used to boost the signal strength in both wired and wireless networks.

---

#### SFP and SFP+

SFP stands for "Small Form-factor Pluggable." It's a type of transceiver that plugs into the SFP port of a network switch to convert the switch's electrical signals into optical signals for transmission over fiber optic cables. SFP modules are hot-pluggable, which means they can be inserted or removed without turning off the network device.

SFP modules can support different communication standards, such as Gigabit Ethernet, Fiber Channel, or SONET, and can be used with different types of fiber, such as single-mode or multi-mode, depending on the specific module. This flexibility makes SFPs a popular choice for many network setups.

SFP+ is an enhanced version of the SFP that supports data rates up to 10 Gbps (Gigabits per second), compared to 1 Gbps for standard SFP modules. Hence, SFP+ is often used in situations where higher data transfer rates are required, such as in data centers or in enterprise-level networking environments.

Both SFP and SFP+ come in various types to support different wavelengths and distances. The right type to use depends on the specific requirements of your network. For example, if you need to transmit data over a long distance, you might use a single-mode SFP or SFP+, while for shorter distances, a multi-mode SFP or SFP+ might be sufficient.

---

#### EMI

EMI stands for Electromagnetic Interference. It's a disturbance that affects an electrical circuit due to either electromagnetic induction or electromagnetic radiation emitted from an external source.

In the context of networking, EMI can cause serious problems. When networking cables, for instance, are exposed to strong EMI, the data signals they carry can become distorted or lost, leading to a decrease in network performance, increased error rates, or even complete loss of network connectivity.

Sources of EMI can include many things: power lines, fluorescent lights, wireless transmitters, computers and other electronic devices, and even motors and other types of heavy machinery.

To help reduce the impact of EMI, networking cables are often shielded, which involves surrounding the cable with a protective material that helps to block electromagnetic fields. For example, shielded twisted pair (STP) cables have a protective shield around each pair of wires, which can help to reduce EMI.

In wireless networking, EMI can also be a problem, as it can cause interference with the wireless signals. This can be mitigated through careful network design, such as selecting the right wireless channels to avoid known sources of interference.

---



