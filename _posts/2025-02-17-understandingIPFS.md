---
layout: post
title: "Understanding IPFS"
date: 2025-02-17
categories: [web3, security, learning]
---


IPFS is a peer-to-peer, distributed file storage protocol designed to make the web faster, safer, and more open. Unlike traditional web systems that retrieve files from centralized servers using URLs, IPFS retrieves content based on what it is rather than where it is.

## Core Concepts of IPFS

1.  **Content Addressing (CID - Content Identifier)**
    * IPFS identifies files by their content hash instead of URLs, ensuring immutability and data integrity.
    * Example CID: `QmXnnyufdzAWLrqzK5jY1jtHKnAybzG1xQumvpmqf7Koab`

2.  **Distributed Hash Table (DHT)**
    * The DHT is a key-value store that helps nodes locate files efficiently in the network.

3.  **Merkle DAG (Directed Acyclic Graph)**
    * IPFS organizes data using a Merkle DAG, where each node stores data and links to its children via CIDs.

4.  **Pinning**
    * Since IPFS is distributed, data may be lost unless pinned for persistence. Tools like Pinata, Web3.Storage, and Filebase offer reliable pinning services.

5.  **IPFS Gateway**
    * An IPFS gateway allows HTTP-based access to IPFS content without installing an IPFS node.
    * Example Gateway URL: `https://ipfs.io/ipfs/QmXnnyufdzAWLrqzK5jY1jtHKnAybzG1xQumvpmqf7Koab`

6.  **IPNS (InterPlanetary Name System)**
    * IPNS references mutable content, acting like a domain name that always points to the latest version of the file.
    * Example IPNS Address: `/ipns/k51qzi5uqu5dlk3x5v0zmbakmz0`

## Problems it solves

* **Verifiability**
    * IPFS uses cryptographic hashes to verify the authenticity and integrity of files, making it difficult for malicious actors to tamper with or delete files.

* **Resilience**
    * IPFS has no single point of failure, and users do not need to trust each other. The failure of a single or even multiple nodes in the network does not affect the functioning of the entire network, and an IPFS node can fetch data from the network as long as at least one other node in the network has that data, regardless of its location.

* **Centralization**
    * IPFS is an open, distributed and participatory network that reduces data silos from centralized servers, making IPFS more resilient than traditional systems. No single entity or person controls, manages or owns IPFS; rather, it is a community-maintained project with multiple implementations of the protocol, multiple tools and apps leveraging that protocol, and multiple users and organizations contributing to its design and development.

* **Performance**
    * IPFS provides faster access to data by enabling it to be replicated to and retrieved from multiple locations, and allowing users to access data from the nearest location using content addressing instead of location-based addressing. Because data can be addressed based on its contents, a node on the network can fetch that data from any other node in the network that has the data; thus, performance issues like latency are reduced.

* **Link rot**
    * IPFS eliminates the problem of link rot by allowing data to be addressed by its content, rather than by its location. Content in IPFS is still reachable regardless of its location, and does not depend on specific servers being available.

* **Data sovereignty**
    * IPFS protects data sovereignty by enabling users to store and access data directly on a decentralized network of nodes, rather than centralized, third-party servers. This eliminates the need for intermediaries to control and manage data, giving users full control and ownership over their data.

* **Off-chain storage**
    * IPFS enables verifiable off-chain storage by creating a link between blockchain state and content-addressed published to IPFS. This works by storing a Content IDentifier (CID) in a smart contract.

* **Local-first software**
    * IPFS benefits local-first software by providing a performant, decentralized, peer-to-peer data addressing, routing, and transfer protocol that prioritizes data storage and processing on individual devices. With IPFS, data can be stored, verified and processed locally, and then synchronized and shared with other IPFS nodes when a network connection is available.

* **Vendor lock-in**
    * IPFS prevents vendor lock-in, as users have sovereignty over their data and infrastructure. This is enabled by content-addressing, which decouples the data from a single location or infrastructure provider. Unlike traditional cloud vendors, IPFS enables you to change data storage locations without changing things like APIs and data management. In addition, because IPFS is open-source, community-maintained and modular, users are not obligated to use a particular subsystem. Instead, users can customize IPFS for their preferred technologies, needs and values.

<!-- ## How it works?
will be written soon ..
🚀 IPFS is revolutionizing decentralized storage! 🚀 -->