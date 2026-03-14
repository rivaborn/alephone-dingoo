# Source_Files/XML/XML_ParseTreeRoot.h

## File Purpose
Header declaring the absolute root element of the XML parser tree and providing initialization/reset routines for the parsing system. Acts as the top-level public interface for XML parsing in the engine.

## Core Responsibilities
- Declare the global `RootParser` as the absolute root element containing all valid XML file roots
- Provide initialization routine to set up the complete parse tree hierarchy
- Provide reset routine to restore all parser values to hardcoded defaults
- Serve as the central entry point for XML document parsing infrastructure

## Key Types / Data Structures
None defined in this file. `XML_ElementParser` is declared in the included header.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| RootParser | XML_ElementParser | extern (global) | Absolute root element of the XML parser tree; contains root elements of all valid XML files |

## Key Functions / Methods

### SetupParseTree
- Signature: `extern void SetupParseTree();`
- Purpose: Initialize and populate the complete XML parse tree, including the root element and all child element parsers
- Inputs: None
- Outputs/Return: None (void)
- Side effects: Constructs the parse tree hierarchy; memory allocation for child parsers
- Calls: Not inferable from this file (implementation in corresponding .cpp)
- Notes: Must be called before any XML parsing operations; initializes the tree structure once per session

### ResetAllMMLValues
- Signature: `extern void ResetAllMMLValues();`
- Purpose: Reset all modified values throughout the parser tree back to hardcoded defaults
- Inputs: None
- Outputs/Return: None (void)
- Side effects: Restores `RootParser` and all child parser state to defaults
- Calls: Not inferable from this file (implementation in corresponding .cpp)
- Notes: Complements `SetupParseTree()`; called during cleanup or state reset operations

## Control Flow Notes
Represents initialization and shutdown hooks for XML parsing:
- `SetupParseTree()` called during engine startup to prepare parsing infrastructure
- `RootParser` serves as central dispatch for all subsequent XML element parsing throughout engine lifetime
- `ResetAllMMLValues()` called during shutdown or when returning engine to default state

## External Dependencies
- **Includes**: `XML_ElementParser.h` ΓÇö defines the `XML_ElementParser` class (base parser with child management and attribute/string handling)
- **External symbols**: Implementations of both functions (defined elsewhere, presumably XML_ParseTreeRoot.cpp)
