# Source_Files/Network/network_distribution_types.h

## File Purpose
Centralized header defining enumerated distribution type IDs for network audio in the Aleph One engine. Provides constants used by the network distribution system to identify different styles of audio packet serialization, supporting both legacy and modern network protocols.

## Core Responsibilities
- Define distribution type identifiers for network audio (`kOriginalNetworkAudioDistributionTypeID`, `kNewNetworkAudioDistributionTypeID`)
- Ensure backward compatibility by distinguishing original vs. new-style network audio formats
- Centralize constant definitions to prevent ID conflicts across the networking subsystem
- Gate definitions behind `DISABLE_NETWORKING` compile-time flag for conditional compilation

## Key Types / Data Structures
None (anonymous enum with integer constants only).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| kOriginalNetworkAudioDistributionTypeID | enum constant (int 0) | global | Legacy network audio format ID; used for compatibility with older client/server versions |
| kNewNetworkAudioDistributionTypeID | enum constant (int 1) | global | Modern realtime network audio format ID; used for current-generation audio streaming |

## Key Functions / Methods
None.

## Control Flow Notes
This is a pure declaration file with no control flow. Preprocessing conditionals (`#ifndef`, `#if !defined(DISABLE_NETWORKING)`) determine whether the header content is available based on build configuration.

## External Dependencies
- **Includes:** `config.h` (build configuration flags)
- **Referenced symbols:** `DISABLE_NETWORKING` macro (conditionally disables entire networking subsystem)
- **Used by:** `NetDistributeInformation()`, `NetAddDistributionFunction()` (defined elsewhere; use these enum values to register/identify audio distribution handlers)
