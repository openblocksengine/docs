
# Oengine: Decentralized Block Space Auction & Block Builder

## Overview

The goal of the project is to create a distributed open auction marketplace and a block builder that makes it easier for validators to maximize revenue from their blocks, and for searchers to efficiently access block space without burning 99% of their profits on tips.

## Problem Statement

Today's Solana block-building ecosystem suffers from opaque, centralized systems dominated by private relays and closed bidding mechanisms. This lack of transparency forces searchers to overpay for inclusion, while validators have no verifiable way to prove fair ordering or earn optimal rewards. At the same time, users face fragmented submission paths (RPCs, TPUs, relays, block engine) that increase network congestion, and stakers are missing out on possible tips.

## Solution

Oengine addresses these issues by introducing open, English-style auctions that begin before slot openings—allowing searchers to submit bids or transactions for upcoming slots in a fair and transparent way. The system runs per-leader-slot auctions, with live, signed updates showing top bids and cutoffs, ensuring everyone receives the same price signal. Winners are determined just before the leader's slot and ordered by bid value, subject to account-lock feasibility.

### Network Layer Architecture

At the network layer, Oengine uses a router-first architecture that connects directly to upcoming leaders. A lightweight gossip/QUIC-based router forwards bids, transactions, and bundles to the appropriate validator, reducing TPU spam and smoothing latency. These messages are compact, signed, and verifiable—forming an auditable off-chain trail of auction activity.

### Fairness and Accountability

To ensure fairness and accountability, validators running Oengine perform on-chain enforcement and slashing. Each validator must sign its auction's final state and produce a block with a minimum match threshold (e.g., 70%) between winning bids and included transactions. If a validator fails to meet the threshold, it is automatically slashed and temporarily blacklisted from running auctions.

### Policy Layer

With a simple, configurable policy layer, validators can adjust parameters like MIN_BID, match thresholds, and ban durations, or enable commit-reveal bidding to prevent last-second sniping. Oengine can also partition block space (e.g., 60% auction / 40% regular flow) to protect regular users while maintaining MEV efficiency.

### Integration

Integration is clean and minimal. The validator only needs a small patch to merge auction winners before TPU flow in the block-building stage, with optional support for future upgrades such as microslot batching for network efficiency.

### Developer Interface

Finally, Oengine provides a developer-friendly interface: a unified CLI for operational tasks—starting or closing auctions, publishing results, uploading Merkle roots, setting policy—and a live auction feed anyone can subscribe to.

## Impact

By combining open auctions, transparent tip flows, validator enforcement, and modular integration, Oengine builds a decentralized, auditable, and incentive-aligned block-building layer for Solana. It not only solves the issues of inefficient price discovery and unreliable inclusion guarantees, but also opens up entirely new classes of searcher products—such as predictive bidding, programmable order flow, and shared execution markets—ushering in a fairer and more decentralized era of block space allocation.

## Technical Components

OEngine consists of four main components:

### 1. Auction Engine: English-Style Block Auction

The auction engine runs on the validator node and conducts an English-style auction for each block during the validator's leader slot. In an English auction, bids are open, and the highest bid wins when the auction closes. When applied to blocks, this means multiple transactions (bids) can win up to the block's capacity, with transactions ordered by bid amount. The auction timing is closely synchronized with Solana's slot schedule.

#### Auction Start

The auction opens one slot before the validator's leadership slot. For example, if the validator is scheduled for leader slot 101, the auction opens near the end of slot 99, just before slot 100 begins. Opening slightly before the previous slot helps ensure the network is aware of the auction as early as possible. A broadcasted auction_start event is emitted to notify bidders.

#### Bidding Phase

During the slot immediately preceding the leader slot (slot 100 in this example), clients submit transactions as bids. Each bid transaction must include a transfer instruction to the validator—either as a direct Lamport transfer to a designated validator address or by calling an on-chain auction program. This embedded payment defines the bid's value.

For simplicity, the design can use a standard Solana transfer instruction to the validator's account within the transaction to serve as the bid fee. As bids arrive, the auction engine updates the list of highest bids in real time. Since this is an English auction, new bids may raise the cutoff for inclusion. The engine broadcasts updates, such as a new highest bid or other significant changes, so all participants receive live feedback on the auction's state. (Clients can subscribe to this live auction feed.)

#### Auction Close

The auction concludes just before the leader's slot starts. At closing, the engine finalizes the winners by sorting all received bid transactions in descending order based on bid amount. The engine may enforce a cap on auction usage; the top bids up to that cap are declared winners, and all other bids are considered losers. The auction_close event is then broadcast to signal the end of the bidding phase.

### 2. Router: Real-Time Auction Engine and Broadcast

RAG (Router, Auction, Gate) — is the component responsible for managing real-time communication within the auction network.

In our architecture, the router and the auction engine are tightly integrated, effectively functioning as part of the same service. The distinction lies in their roles:

- The auction engine manages logic, bidding, and state.
- The router acts as the networking and interface layer, ensuring efficient, verifiable message flow between participants and validators.

#### Core Responsibilities

**1. Accept and Route Transactions**
The router receives bid transactions through high-speed interfaces such as RPC or QUIC. It determines which auction each bid targets—usually the next upcoming auction based on the current slot. If the corresponding auction is open, the bid is immediately added to that auction's pool. This ensures minimal latency and accurate slot alignment for every submitted bid or bundle.

**2. Broadcast Auction State**
To maintain transparency and uphold the English auction model, the router continuously broadcasts updates on the auction's state. Messages include events such as:

- `auction_start`
- `bid_update` (with top bid or top-N bids)
- `auction_end` (with final results)

These updates form a live auction feed. Using the Solana gossip layer, the router pushes updates as soon as new bids arrive, enabling clients and searchers to react in real time. This mechanism ensures that all participants have access to the same information and that no single party can monopolize visibility into the auction.

#### Design Principles

By managing both bid routing and state broadcasting, the router ensures that the auction operates fairly, efficiently, and transparently. It effectively acts as:

- A public mempool for the auction system.
- A real-time order book feed for block space.

While Solana's native runtime lacks a traditional mempool, prior infrastructure such as Jito's block engine introduced an out-of-band mempool and auction feed. Oengine's router extends this concept into an open and decentralized system — anyone can:

- Listen to the router's live auction feed to track bidding activity.
- Submit bids or bundles directly into ongoing auctions.

This open participation is crucial for decentralization. Although the validator ultimately runs and finalizes the auction, the router ensures that the entire process remains auditable, transparent, and accessible — removing the black-box nature of current private auction systems.

#### Gate Interface (Concept)

The "Gate" represents the signed interface through which router events are broadcast and authenticated. Each gate message includes:

- `pubkey` — router or validator identity
- `auction_start`
- `auction_end`
- `bids`

All messages are signed with the node's public key to prevent spoofing and ensure authenticity across the network.

### 3. OProgram

The **OProgram** is the **on-chain governance and accounting layer** of Oengine.
It enforces protocol-level rules for validator accountability, auction integrity, and reward distribution.

Core responsibilities include:

- **Reward Distribution:** Manages automatic kickbacks to stakers based on configurable validator commission rates. Validators can define a fixed or dynamic commission (e.g., 10%), with the remainder distributed to stakers through a **Merkle-based distributor**.
- **Auction Configuration:** Allows validators to set **minimum or starting bids** (`MIN_BID`) and other auction parameters directly on-chain, ensuring transparent configuration across the network.
- **Validator Accountability:** Maintains a **blacklist registry** for validators that fail to meet auction match thresholds, submit invalid auction states, or exhibit malicious behavior. Blacklisted validators are temporarily restricted from running future auctions.
- **On-Chain Verification:** Provides cryptographic verification for auction results, signed validator states, and payout proofs, ensuring all off-chain auction activity is traceable to verifiable on-chain records.

Through the OProgram, Oengine enforces a **trust-minimized economic layer** that aligns incentives among validators, stakers, and searchers, while preserving auditability and transparency across the system.

### 4. Validator Patch

The **Validator Patch** modifies Solana's validator runtime (via **Agave bindings**) to enable **custom block packing and auction-aware transaction handling**.

This patch allows Oengine to integrate seamlessly into the validator's pipeline without disrupting core functionality.

Key functions include:

- **Custom Block Packing:** Exposes the **leader scheduler** to allow Oengine's auction winners to be prioritized when building blocks. This ensures maximum validator revenue while maintaining fair transaction ordering.
- **Early Transaction Filtering:** Adds read access hooks to inspect incoming transactions **before they are buffered in the scheduler**, enabling the system to drop low-value or non-essential transactions early—saving compute and network resources.
- **Auction-Aware Assembly:** Combines the finalized auction winners with the remaining TPU transaction flow at block build time, guaranteeing inclusion consistency between the auction results and the produced block.
- **Performance Optimization:** Creates a pathway for future improvements such as **microslot batching**, where small transactions can be aggregated to optimize throughput and reduce per-packet overhead.

In essence, the Validator Patch turns a standard Solana validator into an **auction-capable builder node**, capable of maximizing MEV capture transparently while preserving Solana's low-latency performance.
