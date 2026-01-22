# **Case Study: Evolving Beyond UI-Scripts with Agentic Claim Validation**

## **Transforming P\&C Claim Audits from Screen-Scraping to Schema-Driven Intelligence at AgentisIQ**

### **Executive Summary**

This case study details the strategic transformation of a high-volume **Property & Casualty (P\&C) Claim Validation** process. By migrating from traditional RPA scripts that relied on fixed screen coordinates to an **Intelligent AI Agent** architecture, **AgentisIQ** eliminated the "brittle-script" cycle. The new system utilizes the **Model Context Protocol (MCP)** and structured **Claim Schemas** to achieve goal-oriented automation that understands *what* to validate, regardless of *where* the data appears on the screen.

### **1\. The Use Case: P\&C Claim Data Validation**

The core challenge involved validating incoming P\&C insurance claims against policy documents, police reports, and repair estimates.

**The Environment:** Claims adjusters interact with multiple legacy platforms, a web-based claims portal, a green-screen mainframe for policy lookups, and third-party estimate software.

**The Cognitive Requirement:**

* **Contextual Cross-Referencing:** Determining if a "fender bender" in a narrative report matches the "front-end collision" code in the core system.  
* **Dynamic Data Location:** Claim numbers, dates, and amounts shift positions based on the insurance line (Auto vs. Home) or the specific vendor's invoice format.

### **2\. The Problem: The Failure of Fixed UI Elements**

Initially, the client used standard RPA scripts. These scripts were programmed to "Click at X:200, Y:450" to find the claim amount. This led to three systemic failures:

1. **UI Fragility:** A single pixel shift or a minor browser update would break the entire validation flow, requiring immediate manual repair.  
2. **Schema Ignorance:** RPA had no concept of a "Claim Object." If a field was empty, the bot would simply error out because it couldn't find the visual element, rather than understanding that a "Missing Date of Loss" is a validation failure in itself.  
3. **Infrastructure Bloat:** Validating complex claims across three different applications required heavy VDI (Virtual Desktop Infrastructure) resources, capping concurrency at very low levels.

### **3\. The Solution: Goal-Oriented Automation via MCP**

**Role of Srinivasan Sankar:** As the **Lead AI Strategist**, Srinivasan Sankar shifted the automation paradigm from "Step-by-Step UI Execution" to **"Goal-Oriented Validation."**

**Specific Contributions:**

* **Model Context Protocol (MCP) Implementation:** Srinivasan integrated MCP to provide the AI Agent with a standardized way to access "context" across different tools. Instead of teaching the bot how to click, he gave the agent a "Claim Schema", a structured definition of what a valid claim looks like.  
* **Contextual Understanding:** The agent was assigned the goal: **"Validate Claim Data."** Using Vision LLMs and MCP-connected tools, the agent "looks" at the screen to find the required data points based on their semantic meaning (e.g., "Find the deductible") rather than their screen coordinates.  
* **Schema-Driven Reasoning:** By using **AI Agents**, Srinivasan ensured that the output of every validation was strictly typed. The agent doesn't just "scrape text"; it populates a structured data model that enforces business logic (e.g., "Date of Loss cannot be after Date of Filing").  
* **Asynchronous Processing:** He replaced the single-threaded RPA bot with an event-driven agent loop. This allowed the system to handle multiple claim validations simultaneously on a single backend instance, bypassing the need for expensive VDIs.

### **4\. Key Results & Lessons Learned**

#### **The Outcome:**

* **Maintenance Elimination:** Because the agent understands the **Claim Schema**, it successfully processed claims even after the claims portal underwent a major UI redesign, requiring **zero developer intervention**.  
* **Accuracy Boost:** Validation accuracy rose from 82% (RPA) to 99.4% (Agentic), as the AI could interpret unstructured narratives in police reports.  
* **Concurrency:** Achieved a **12x increase in processing capacity** without adding additional server overhead.

#### **What Was Learned:**

1. **Goals \> Steps:** Automation is more resilient when you define the **Goal** (Validate the Claim) and provide the **Context** (The Schema) rather than defining the specific UI steps.  
2. **MCP is the Glue:** The Model Context Protocol is essential for giving agents a "standardized view" of disparate legacy systems, turning fragmented apps into a unified knowledge base.  
3. **Semantic Search is Key:** Agents should "search" for fields visually or via DOM queries based on labels and proximity logic, which is far more robust than static selectors.

### **5\. Conclusion**

The transition at AgentisIQ proved that for P\&C claims, the future of automation isn't better scripts, it's **better context**. By utilizing MCP and schema-driven reasoning, Srinivasan Sankar demonstrated that AI Agents can navigate complex business environments with the same adaptability as a human adjuster, but with the speed and scale of a machine.

© 2026 Srinivasan Sankar @AgentisIQ. All Rights Reserved.