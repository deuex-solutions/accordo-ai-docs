# Comprehensive Seed Data Plan (Final Draft)

## Overview

This document outlines the complete seed data strategy for the Accordo AI procurement negotiation platform. The seed data will cover all modules and test scenarios for end-to-end testing.

---

## Key Clarifications

### L1, L2, L3 = Vendor Offer Rankings (NOT Approval Levels)

| Term | Meaning |
|------|---------|
| **L1** | Lowest priced vendor offer for an RFQ |
| **L2** | Second lowest priced vendor offer |
| **L3** | Third lowest priced vendor offer |

**Calculation Trigger:**
- When negotiation deadline is reached, OR
- When all attached vendors have submitted their final offers
- Whichever happens first

**Scope:** Per total RFQ value (not per product line item)

**Workflow:**
1. Vendors submit offers through negotiation chat
2. System calculates L1/L2/L3 ranking based on total RFQ value
3. Comparison page shows all vendors with full breakdown + weighted score
4. Procurement selects ONE winning vendor (can choose L1, L2, or L3)
5. If selection amount exceeds threshold → Approval workflow triggered
6. After approval → PO generated

### Approval Hierarchy (Renamed from L1/L2/L3)

| Old Name | New Name | Approval Limit |
|----------|----------|----------------|
| L1 Approver | Manager | Up to $10,000 |
| L2 Approver | Director | Up to $50,000 |
| L3 Approver | VP/Executive | Up to $500,000 |

**Trigger:** Amount-based - only high-value selections need approval

---

## Summary of Requirements

| Category | Specification |
|----------|---------------|
| Companies | Medium (15-25): 3-5 enterprise + 12-20 vendors |
| Users per Company | Minimal (2-3): 1 admin + 1-2 users |
| Product Categories | All: IT/Electronics, Office/Furniture, Manufacturing |
| Projects | Moderate (10-15) |
| Requisitions | Basic (10-15) with all statuses |
| Vendors per RFQ | Realistic distribution (1-6 based on RFQ type) |
| Chat Outcomes | All: ACCEPTED, WALKED_AWAY, ESCALATED, NEGOTIATING |
| Chat Depths | Mixed: Short (2-4), Medium (5-10), Long (15-25) |
| Chat Mode | CONVERSATION only |
| Bid Workflow | All stages: Offers → L1/L2/L3 Ranking → Selection → Approval → PO |
| Email Logs | All types: vendor_attached, status_change, reminder, bounced |
| AI/ML Data | Both: Training data + Mock embeddings |
| Price Range | Low value ($100 - $10,000) |
| Date Range | Full year (Jan 2025 - Jan 2026) |
| Naming Style | Mixed (realistic + test data) |
| Idempotent | Yes - findOrCreate pattern |

---

## 1. Companies (20 Total)

### Enterprise Customers (4)
| ID | Company Name | Type | Industry | Currency |
|----|--------------|------|----------|----------|
| 1 | Accordo Technologies | Enterprise | Technology | USD |
| 2 | BuildRight Construction | Enterprise | Construction | USD |
| 3 | MediCore Health Systems | Enterprise | Healthcare | USD |
| 4 | EduFirst Learning Corp | Enterprise | Education | USD |

### Vendor Companies (16)
| ID | Company Name | Sector | Nature | Rating (1-5) |
|----|--------------|--------|--------|--------------|
| 5 | TechSupply Corp | IT/Electronics | Domestic | 4.5 |
| 6 | GlobalParts Inc | IT/Electronics | International | 4.2 |
| 7 | ServerDirect USA | IT/Electronics | Domestic | 4.7 |
| 8 | CloudSoft Solutions | Software | Domestic | 4.3 |
| 9 | OfficeMax Pro | Office Supplies | Domestic | 4.0 |
| 10 | FurniturePlus Ltd | Office Furniture | Domestic | 3.8 |
| 11 | WorkSpace Designs | Office Furniture | International | 4.4 |
| 12 | MetalWorks Global | Manufacturing | International | 4.1 |
| 13 | SteelCraft Industries | Manufacturing | Domestic | 3.9 |
| 14 | AlloySupply Co | Manufacturing | Domestic | 4.0 |
| 15 | PrecisionParts Inc | Manufacturing | Domestic | 4.6 |
| 16 | PackagePro Solutions | Packaging | Domestic | 3.7 |
| 17 | LogiTrans Shipping | Logistics | Domestic | 4.2 |
| 18 | QualityCert Labs | Testing/QA | Domestic | 4.8 |
| 19 | SafetyFirst Equipment | Safety/PPE | Domestic | 4.4 |
| 20 | GreenSupply Eco | Sustainable | International | 4.1 |

---

## 2. Users (60 Total)

### User Distribution by Role

| Role | Count | Password | Email Pattern |
|------|-------|----------|---------------|
| Super Admin | 4 | `Admin@2026!` | admin@{company}.com |
| Procurement Manager | 8 | `Procure@2026!` | procurement@{company}.com |
| Project Manager | 4 | `Project@2026!` | pm@{company}.com |
| Manager (Approver) | 4 | `Manager@2026!` | manager@{company}.com |
| Director (Approver) | 4 | `Director@2026!` | director@{company}.com |
| VP/Executive (Approver) | 4 | `Executive@2026!` | vp@{company}.com |
| Vendor Users | 32 | `Vendor@2026!` | sales@{vendor}.com |

### Enterprise Users - Accordo Technologies (Company 1)

| User | Email | Role | Password | Approval Limit |
|------|-------|------|----------|----------------|
| John Admin | admin@accordo.ai | Super Admin | `Admin@2026!` | - |
| Jane Procurement | procurement@accordo.ai | Procurement Manager | `Procure@2026!` | - |
| Tom Project | pm@accordo.ai | Project Manager | `Project@2026!` | - |
| Sarah Manager | manager@accordo.ai | Manager (Approver) | `Manager@2026!` | $10,000 |
| Mike Director | director@accordo.ai | Director (Approver) | `Director@2026!` | $50,000 |
| Lisa VP | vp@accordo.ai | VP/Executive (Approver) | `Executive@2026!` | $500,000 |

*Similar structure for BuildRight, MediCore, EduFirst*

### Vendor Users (2 per vendor company)

| Vendor Company | User 1 (Email / Password) | User 2 (Email / Password) |
|----------------|---------------------------|---------------------------|
| TechSupply Corp | sales@techsupply.com / `Vendor@2026!` | accounts@techsupply.com / `Vendor@2026!` |
| GlobalParts Inc | sales@globalparts.eu / `Vendor@2026!` | orders@globalparts.eu / `Vendor@2026!` |
| ServerDirect USA | sales@serverdirect.us / `Vendor@2026!` | support@serverdirect.us / `Vendor@2026!` |
| CloudSoft Solutions | sales@cloudsoft.io / `Vendor@2026!` | licenses@cloudsoft.io / `Vendor@2026!` |
| OfficeMax Pro | sales@officemaxpro.com / `Vendor@2026!` | orders@officemaxpro.com / `Vendor@2026!` |
| FurniturePlus Ltd | sales@furnitureplus.com / `Vendor@2026!` | design@furnitureplus.com / `Vendor@2026!` |
| WorkSpace Designs | sales@workspacedesigns.eu / `Vendor@2026!` | projects@workspacedesigns.eu / `Vendor@2026!` |
| MetalWorks Global | sales@metalworksglobal.com / `Vendor@2026!` | exports@metalworksglobal.com / `Vendor@2026!` |
| SteelCraft Industries | sales@steelcraft.in / `Vendor@2026!` | orders@steelcraft.in / `Vendor@2026!` |
| AlloySupply Co | sales@alloysupply.com / `Vendor@2026!` | quotes@alloysupply.com / `Vendor@2026!` |
| PrecisionParts Inc | sales@precisionparts.com / `Vendor@2026!` | engineering@precisionparts.com / `Vendor@2026!` |
| PackagePro Solutions | sales@packagepro.com / `Vendor@2026!` | logistics@packagepro.com / `Vendor@2026!` |
| LogiTrans Shipping | sales@logitrans.com / `Vendor@2026!` | tracking@logitrans.com / `Vendor@2026!` |
| QualityCert Labs | sales@qualitycert.com / `Vendor@2026!` | testing@qualitycert.com / `Vendor@2026!` |
| SafetyFirst Equipment | sales@safetyfirst.com / `Vendor@2026!` | orders@safetyfirst.com / `Vendor@2026!` |
| GreenSupply Eco | sales@greensupply.eco / `Vendor@2026!` | sustainability@greensupply.eco / `Vendor@2026!` |

---

## 3. Products (50 Total)

### IT & Electronics (15 products)
| ID | Product Name | Category | UOM | Price Range |
|----|--------------|----------|-----|-------------|
| 1 | Dell XPS 15 Laptop | Electronics | unit | $1,200 - $1,800 |
| 2 | Dell PowerEdge R750 Server | Servers | unit | $5,000 - $8,000 |
| 3 | Cisco Catalyst 9300 Switch | Networking | unit | $3,000 - $5,000 |
| 4 | HP LaserJet Pro Printer | Printers | unit | $400 - $600 |
| 5 | Microsoft 365 E5 License | Software | license | $40 - $60/user |
| 6 | Adobe Creative Cloud | Software | license | $50 - $80/user |
| 7 | Slack Business+ | Software | license | $12 - $18/user |
| 8 | 27" Dell Monitor | Displays | unit | $300 - $500 |
| 9 | Logitech Webcam C930e | Peripherals | unit | $100 - $150 |
| 10 | Mechanical Keyboard | Peripherals | unit | $80 - $150 |
| 11 | 1TB SSD Drive | Storage | unit | $100 - $200 |
| 12 | 32GB RAM Module | Components | unit | $150 - $250 |
| 13 | CAT6 Ethernet Cable (100ft) | Cables | roll | $30 - $50 |
| 14 | UPS Battery Backup 1500VA | Power | unit | $200 - $350 |
| 15 | Security Camera System | Security | unit | $500 - $1,000 |

### Office & Furniture (15 products)
| ID | Product Name | Category | UOM | Price Range |
|----|--------------|----------|-----|-------------|
| 16 | Ergonomic Office Chair | Furniture | unit | $300 - $600 |
| 17 | Standing Desk 60" | Furniture | unit | $400 - $800 |
| 18 | Conference Table 12-seat | Furniture | unit | $1,500 - $3,000 |
| 19 | Filing Cabinet 4-drawer | Storage | unit | $200 - $400 |
| 20 | Whiteboard 6x4 ft | Office | unit | $150 - $300 |
| 21 | A4 Paper (Box of 5000) | Supplies | box | $40 - $60 |
| 22 | Printer Toner Multipack | Supplies | pack | $150 - $250 |
| 23 | Office Desk Lamp LED | Lighting | unit | $50 - $100 |
| 24 | Paper Shredder | Equipment | unit | $100 - $200 |
| 25 | Water Dispenser | Appliance | unit | $200 - $400 |
| 26 | Coffee Machine Commercial | Appliance | unit | $500 - $1,000 |
| 27 | First Aid Kit | Safety | kit | $50 - $100 |
| 28 | Fire Extinguisher | Safety | unit | $80 - $150 |
| 29 | Office Plant Set | Decor | set | $100 - $200 |
| 30 | Wall Clock Digital | Decor | unit | $30 - $60 |

### Manufacturing & Raw Materials (20 products)
| ID | Product Name | Category | UOM | Price Range |
|----|--------------|----------|-----|-------------|
| 31 | Steel Coil Grade A | Raw Metal | ton | $800 - $1,200 |
| 32 | Aluminum Sheet 6061 | Raw Metal | sheet | $50 - $100 |
| 33 | Copper Wire AWG 10 | Electrical | roll | $80 - $150 |
| 34 | Stainless Steel Pipe | Piping | meter | $20 - $40 |
| 35 | Industrial Lubricant | Chemicals | liter | $15 - $30 |
| 36 | Welding Rods (Pack 100) | Consumables | pack | $30 - $60 |
| 37 | Safety Goggles | PPE | unit | $20 - $40 |
| 38 | Work Gloves Industrial | PPE | pair | $15 - $30 |
| 39 | Hard Hat OSHA | PPE | unit | $25 - $50 |
| 40 | Steel Toe Boots | PPE | pair | $80 - $150 |
| 41 | Packaging Boxes (100pc) | Packaging | pack | $50 - $100 |
| 42 | Bubble Wrap Roll | Packaging | roll | $30 - $60 |
| 43 | Pallet Jack Manual | Equipment | unit | $300 - $500 |
| 44 | Forklift Battery | Power | unit | $500 - $1,000 |
| 45 | Conveyor Belt Section | Equipment | meter | $100 - $200 |
| 46 | CNC Tool Bits Set | Tools | set | $200 - $400 |
| 47 | Measuring Calipers | Tools | unit | $50 - $100 |
| 48 | Industrial Fan 24" | HVAC | unit | $150 - $300 |
| 49 | Air Compressor 5HP | Equipment | unit | $800 - $1,500 |
| 50 | Dust Collector System | Safety | unit | $1,000 - $2,000 |

---

## 4. Projects (12 Total)

| ID | Project Name | Company | Type | Duration |
|----|--------------|---------|------|----------|
| 1 | IT Infrastructure Upgrade 2026 | Accordo | IT | 180 days |
| 2 | Office Expansion Phase 1 | Accordo | Construction | 120 days |
| 3 | Software License Renewal | Accordo | IT | 30 days |
| 4 | Data Center Migration | Accordo | IT | 240 days |
| 5 | Hospital Wing Renovation | MediCore | Construction | 365 days |
| 6 | Medical Equipment Procurement | MediCore | Healthcare | 90 days |
| 7 | Campus Security Upgrade | EduFirst | Security | 60 days |
| 8 | Lab Equipment Refresh | EduFirst | Education | 45 days |
| 9 | Construction Site Setup | BuildRight | Construction | 90 days |
| 10 | Safety Equipment Annual | BuildRight | Safety | 30 days |
| 11 | Manufacturing Line Upgrade | Accordo | Manufacturing | 150 days |
| 12 | Green Initiative Supplies | Accordo | Sustainability | 60 days |

---

## 5. Requisitions (15 Total)

### Status Distribution
| Status | Count | Description |
|--------|-------|-------------|
| Draft | 2 | Not yet submitted |
| Created | 2 | Submitted, awaiting vendors |
| NegotiationStarted | 4 | Active negotiations |
| Fulfilled | 2 | Completed successfully |
| Awarded | 2 | Vendor selected, PO issued |
| Cancelled | 1 | Cancelled by user |
| Expired | 2 | Deadline passed |

### Requisition Details with Vendor Distribution

| RFQ ID | Project | Subject | Total Value | Status | Vendors | L1/L2/L3 Status |
|--------|---------|---------|-------------|--------|---------|-----------------|
| RFQ0001 | IT Infrastructure | Laptop Procurement Q1 | $8,500 | NegotiationStarted | 4 | Pending |
| RFQ0002 | Office Expansion | Office Furniture Set | $6,200 | NegotiationStarted | 5 | Pending |
| RFQ0003 | Software Renewal | Annual Licenses | $4,800 | Awarded | 3 | L1 Selected |
| RFQ0004 | Data Center | Server Hardware | $9,500 | NegotiationStarted | 3 | Pending |
| RFQ0005 | Hospital Wing | Medical Supplies | $7,200 | Fulfilled | 4 | L2 Selected (faster delivery) |
| RFQ0006 | Medical Equipment | Diagnostic Tools | $5,500 | NegotiationStarted | 6 | Pending |
| RFQ0007 | Campus Security | CCTV System | $3,800 | Created | 2 | Not Yet |
| RFQ0008 | Lab Equipment | Lab Furniture | $4,200 | Draft | 0 | N/A |
| RFQ0009 | Construction Site | Safety Equipment | $2,800 | Awarded | 3 | L1 Selected |
| RFQ0010 | Safety Equipment | PPE Annual Order | $1,900 | Fulfilled | 2 | L1 Selected |
| RFQ0011 | Manufacturing | Raw Materials Q1 | $8,900 | Expired | 5 | L3 Selected (best rating) |
| RFQ0012 | Green Initiative | Eco-Friendly Supplies | $3,200 | Draft | 0 | N/A |
| RFQ0013 | IT Infrastructure | Network Equipment | $7,600 | Cancelled | 3 | Cancelled |
| RFQ0014 | Office Expansion | Appliances Set | $2,400 | Expired | 2 | Only L1, L2 available |
| RFQ0015 | Data Center | Storage Systems | $9,800 | Created | 4 | Not Yet |

### Scoring Weights (Configurable per RFQ)

Each RFQ will have configurable weights for the 6 scoring factors:

| Factor | Default Weight | Description |
|--------|----------------|-------------|
| Price | 35% | Total quoted price |
| Delivery Time | 20% | Days to deliver |
| Payment Terms | 15% | Net payment days |
| Vendor Rating | 15% | Overall 1-5 star rating |
| Past Performance | 10% | Historical on-time delivery % |
| Quality Certifications | 5% | ISO, CE, etc. compliance |

**Example Weight Configurations in Seed Data:**
- RFQ0001 (Laptops): Price 40%, Delivery 25%, Terms 15%, Rating 10%, Performance 5%, Quality 5%
- RFQ0005 (Medical): Price 25%, Delivery 30%, Terms 10%, Rating 15%, Performance 10%, Quality 10%
- RFQ0011 (Raw Materials): Price 30%, Delivery 15%, Terms 20%, Rating 20%, Performance 10%, Quality 5%

---

## 6. L1/L2/L3 Offer Scenarios (Test Cases)

### Scenario 1: Clear Winner (RFQ0003)
| Rank | Vendor | Total Price | Delivery | Terms | Rating | Score |
|------|--------|-------------|----------|-------|--------|-------|
| L1 | CloudSoft | $4,650 | 14 days | Net 30 | 4.3 | 87.5 |
| L2 | TechSupply | $4,850 | 21 days | Net 45 | 4.5 | 79.2 |
| L3 | GlobalParts | $5,100 | 28 days | Net 60 | 4.2 | 71.8 |

**Selection:** L1 (CloudSoft) - Best price and score
**Reason:** "Lowest price with acceptable delivery timeline"

### Scenario 2: Close Competition (RFQ0009)
| Rank | Vendor | Total Price | Delivery | Terms | Rating | Score |
|------|--------|-------------|----------|-------|--------|-------|
| L1 | SafetyFirst | $2,650 | 10 days | Net 30 | 4.4 | 84.2 |
| L2 | PrecisionParts | $2,720 | 7 days | Net 30 | 4.6 | 83.8 |
| L3 | AlloySupply | $2,780 | 12 days | Net 45 | 4.0 | 78.5 |

**Selection:** L1 (SafetyFirst) - Marginal win
**Reason:** "Best overall value despite close scoring"

### Scenario 3: Trade-off Selection (RFQ0005)
| Rank | Vendor | Total Price | Delivery | Terms | Rating | Score |
|------|--------|-------------|----------|-------|--------|-------|
| L1 | PackagePro | $6,900 | 21 days | Net 60 | 3.7 | 76.3 |
| L2 | SafetyFirst | $7,100 | 7 days | Net 30 | 4.4 | 82.1 |
| L3 | QualityCert | $7,450 | 14 days | Net 45 | 4.8 | 79.6 |

**Selection:** L2 (SafetyFirst) - NOT the lowest price
**Reason:** "Medical supplies require faster delivery. L2 offers 7-day delivery vs L1's 21 days. Higher score due to delivery weight."

### Scenario 4: Rating-Based Selection (RFQ0011)
| Rank | Vendor | Total Price | Delivery | Terms | Rating | Score |
|------|--------|-------------|----------|-------|--------|-------|
| L1 | SteelCraft | $8,500 | 30 days | Net 45 | 3.9 | 74.2 |
| L2 | MetalWorks | $8,700 | 28 days | Net 30 | 4.1 | 76.8 |
| L3 | PrecisionParts | $8,950 | 21 days | Net 30 | 4.6 | 81.5 |

**Selection:** L3 (PrecisionParts) - Highest rated vendor
**Reason:** "Critical manufacturing parts require reliability. L3's higher rating and past performance outweigh $450 price difference."

### Scenario 5: Edge Case - Only 2 Vendors (RFQ0014)
| Rank | Vendor | Total Price | Delivery | Terms | Rating | Score |
|------|--------|-------------|----------|-------|--------|-------|
| L1 | FurniturePlus | $2,280 | 14 days | Net 30 | 3.8 | 78.4 |
| L2 | WorkSpace | $2,520 | 10 days | Net 45 | 4.4 | 76.9 |
| L3 | - | - | - | - | - | - |

**Selection:** Expired before selection
**Reason:** "Deadline reached with only 2 vendors. Insufficient competition."

### Scenario 6: Single Vendor (RFQ0010)
| Rank | Vendor | Total Price | Delivery | Terms | Rating | Score |
|------|--------|-------------|----------|-------|--------|-------|
| L1 | SafetyFirst | $1,820 | 5 days | Net 30 | 4.4 | 85.0 |
| L2 | - | - | - | - | - | - |
| L3 | - | - | - | - | - | - |

**Selection:** L1 (SafetyFirst) - Only option
**Reason:** "Single vendor submission. Price within budget, proceed with L1."

---

## 7. Contracts (42 Total)

Based on vendor distribution across requisitions:

| RFQ | Vendors Attached | Contracts Created |
|-----|------------------|-------------------|
| RFQ0001 | 4 | 4 |
| RFQ0002 | 5 | 5 |
| RFQ0003 | 3 | 3 |
| RFQ0004 | 3 | 3 |
| RFQ0005 | 4 | 4 |
| RFQ0006 | 6 | 6 |
| RFQ0007 | 2 | 2 |
| RFQ0008 | 0 | 0 (Draft) |
| RFQ0009 | 3 | 3 |
| RFQ0010 | 2 | 2 |
| RFQ0011 | 5 | 5 |
| RFQ0012 | 0 | 0 (Draft) |
| RFQ0013 | 3 | 3 |
| RFQ0014 | 2 | 2 |
| RFQ0015 | 4 | 4 |
| **Total** | **46** | **42** |

Each contract includes:
- Unique 32-char token for vendor portal access
- Status: Created → Opened → Completed/Rejected
- Link to ChatbotDeal (for negotiation)
- Final offer details when completed

---

## 8. Negotiation Chats (42 Total - One per Contract)

### Chat Distribution by Outcome

| Outcome | Count | Description |
|---------|-------|-------------|
| ACCEPTED | 14 | Successful deal closure |
| WALKED_AWAY | 10 | Utility below threshold |
| ESCALATED | 8 | Max rounds reached or manual |
| NEGOTIATING | 10 | Still in progress |

### Chat Depth Distribution

| Depth | Messages | Count |
|-------|----------|-------|
| Short | 2-4 | 10 |
| Medium | 5-10 | 20 |
| Long | 15-25 | 12 |

### Sample Chat Conversations

**Chat 1: Quick Accept (Short - 4 messages)**
```
[ACCORDO] Welcome to the negotiation for Laptop Procurement Q1.
          Our target price is $8,200 for 5 units.

[VENDOR]  Thank you. We can offer $8,800 with 14-day delivery and Net 30 terms.

[ACCORDO] We appreciate the offer. Can you come down to $8,400?
          That's within our budget range.

[VENDOR]  We can meet you at $8,500 with the same terms. Final offer.

[SYSTEM]  Deal ACCEPTED. Final price: $8,500. Utility score: 0.82
```

**Chat 2: Extended Negotiation (Long - 18 messages)**
```
Round 1: Vendor offers $7,500 → Accordo counters $6,200
Round 2: Vendor offers $7,200 → Accordo counters $6,400
Round 3: Vendor offers $7,000 → Accordo counters $6,600
Round 4: Vendor offers $6,850 → Accordo counters $6,700
Round 5: Vendor offers $6,780 → Accordo counters $6,720
Round 6: Vendor offers $6,750 → ACCEPTED

[SYSTEM]  Deal ACCEPTED after 6 rounds. Final price: $6,750. Utility: 0.78
```

**Chat 3: Walk Away (Medium - 8 messages)**
```
Round 1: Vendor offers $5,800 → Accordo counters $4,200
Round 2: Vendor offers $5,600 → Accordo counters $4,500
Round 3: Vendor offers $5,500 (final) → Accordo calculates utility: 0.38
Round 4: Accordo message: "Unfortunately, this offer is below our acceptable threshold."

[SYSTEM]  Deal WALKED_AWAY. Vendor's final offer: $5,500. Utility: 0.38 (below 0.45 threshold)
```

**Chat 4: Escalation (Medium - 12 messages)**
```
Round 1-5: Back and forth negotiation on complex terms
Round 6: Vendor requests custom payment schedule
Round 7: Accordo unable to process non-standard terms

[SYSTEM]  Deal ESCALATED. Reason: Complex payment terms require human review.
          Current offer: $4,200 with custom payment schedule.
```

---

## 9. Vendor Bids (42 Total)

| Bid Status | Count | Description |
|------------|-------|-------------|
| PENDING | 10 | Negotiation in progress |
| COMPLETED | 18 | Negotiation finished, awaiting comparison |
| SELECTED | 6 | Won the bid |
| REJECTED | 6 | Lost to another vendor |
| EXCLUDED | 2 | Walked away from deal |

Each VendorBid record includes:
- `finalPrice`: Total quoted price
- `unitPrice`: Per-unit price
- `paymentTerms`: Net 30/45/60
- `deliveryDate`: Promised delivery
- `utilityScore`: 0-1 score from negotiation
- `chatSummaryMetrics`: JSON with rounds, price changes, etc.
- `chatSummaryNarrative`: AI-generated summary

---

## 10. Bid Comparisons (8 Total)

| ID | RFQ | Trigger | Vendors | L1 | L2 | L3 |
|----|-----|---------|---------|----|----|----|
| 1 | RFQ0003 | ALL_COMPLETED | 3 | CloudSoft ($4,650) | TechSupply ($4,850) | GlobalParts ($5,100) |
| 2 | RFQ0005 | ALL_COMPLETED | 4 | PackagePro ($6,900) | SafetyFirst ($7,100) | QualityCert ($7,450) |
| 3 | RFQ0009 | DEADLINE_REACHED | 3 | SafetyFirst ($2,650) | PrecisionParts ($2,720) | AlloySupply ($2,780) |
| 4 | RFQ0010 | ALL_COMPLETED | 2 | SafetyFirst ($1,820) | - | - |
| 5 | RFQ0011 | DEADLINE_REACHED | 5 | SteelCraft ($8,500) | MetalWorks ($8,700) | PrecisionParts ($8,950) |
| 6 | RFQ0014 | DEADLINE_REACHED | 2 | FurniturePlus ($2,280) | WorkSpace ($2,520) | - |
| 7 | RFQ0001 | MANUAL | 4 | Pending | Pending | Pending |
| 8 | RFQ0006 | MANUAL | 6 | Pending | Pending | Pending |

Each comparison includes:
- `topBidsJson`: Array of ranked vendor offers
- `pdfUrl`: Generated comparison report
- Weighted scores for each vendor

---

## 11. Vendor Selections & Approvals (6 Total)

| Selection | RFQ | Winning Vendor | Selected Rank | Price | Approval Required | Reason |
|-----------|-----|----------------|---------------|-------|-------------------|--------|
| 1 | RFQ0003 | CloudSoft | L1 | $4,650 | No (< $10K) | "Lowest price with acceptable terms" |
| 2 | RFQ0005 | SafetyFirst | L2 | $7,100 | No (< $10K) | "Faster delivery critical for medical supplies" |
| 3 | RFQ0009 | SafetyFirst | L1 | $2,650 | No (< $10K) | "Best overall score" |
| 4 | RFQ0010 | SafetyFirst | L1 | $1,820 | No (< $10K) | "Single vendor, price within budget" |
| 5 | RFQ0011 | PrecisionParts | L3 | $8,950 | No (< $10K) | "Highest rating and reliability for critical parts" |
| 6 | RFQ0014 | - | - | - | - | "Expired without selection" |

---

## 12. Purchase Orders (5 Total)

| PO Number | RFQ | Vendor | Amount | Status |
|-----------|-----|--------|--------|--------|
| PO-2026-001 | RFQ0003 | CloudSoft | $4,650 | Created |
| PO-2026-002 | RFQ0005 | SafetyFirst | $7,100 | Created |
| PO-2026-003 | RFQ0009 | SafetyFirst | $2,650 | Created |
| PO-2026-004 | RFQ0010 | SafetyFirst | $1,820 | Created |
| PO-2026-005 | RFQ0011 | PrecisionParts | $8,950 | Created |

---

## 13. Email Logs (55 Total)

| Email Type | Count | Description |
|------------|-------|-------------|
| vendor_attached | 25 | When vendors added to RFQ |
| status_change | 15 | Contract status updates |
| reminder | 8 | Deadline reminders |
| bounced | 4 | Failed delivery |
| other | 3 | Miscellaneous |

---

## 14. AI/ML Data

### Negotiation Training Data (25 records)
- Captured from completed negotiations
- Includes: suggestions JSON, conversation context, config snapshot
- Generation source: Mix of 'llm' and 'fallback'
- Selected suggestions tracked for model improvement

### Mock Vector Embeddings

| Type | Count | Description |
|------|-------|-------------|
| MessageEmbedding | 60 | For semantic search |
| DealEmbedding | 25 | For similar deal matching |
| NegotiationPattern | 12 | Common negotiation patterns |

---

## 15. Test Credentials Summary

### Quick Reference - Enterprise Users (Accordo Technologies)

| Role | Email | Password |
|------|-------|----------|
| Super Admin | admin@accordo.ai | `Admin@2026!` |
| Procurement Manager | procurement@accordo.ai | `Procure@2026!` |
| Project Manager | pm@accordo.ai | `Project@2026!` |
| Manager (Approver) | manager@accordo.ai | `Manager@2026!` |
| Director (Approver) | director@accordo.ai | `Director@2026!` |
| VP/Executive (Approver) | vp@accordo.ai | `Executive@2026!` |

### Quick Reference - Vendor Users

| Vendor | Email | Password |
|--------|-------|----------|
| TechSupply | sales@techsupply.com | `Vendor@2026!` |
| GlobalParts | sales@globalparts.eu | `Vendor@2026!` |
| ServerDirect | sales@serverdirect.us | `Vendor@2026!` |
| CloudSoft | sales@cloudsoft.io | `Vendor@2026!` |
| SafetyFirst | sales@safetyfirst.com | `Vendor@2026!` |

---

## 16. Schema Changes Required

### Rename Approval Levels

**Before:**
```typescript
approvalLevel: ENUM('L1', 'L2', 'L3')
```

**After:**
```typescript
approvalLevel: ENUM('MANAGER', 'DIRECTOR', 'VP')
```

**Files to Update:**
- `src/models/user.ts` - User.approvalLevel
- `src/models/requisition.ts` - currentApprovalLevel, requiredApprovalLevel
- `src/models/approval.ts` - approvalLevel
- `src/types/index.ts` - Type definitions
- Migration file to alter existing data

### Add Scoring Weights to Requisition

**New Field:**
```typescript
scoringWeights: {
  type: DataTypes.JSONB,
  allowNull: true,
  defaultValue: {
    price: 35,
    delivery: 20,
    paymentTerms: 15,
    vendorRating: 15,
    pastPerformance: 10,
    qualityCertifications: 5
  }
}
```

### Add Vendor Rating to Company

**New Field:**
```typescript
overallRating: {
  type: DataTypes.DECIMAL(2, 1),
  allowNull: true,
  defaultValue: null,
  validate: { min: 1.0, max: 5.0 }
}
```

---

## 17. Seed Script Structure

```
src/seeders/
├── index.ts                    # Main orchestrator
├── data/
│   ├── companies.ts            # 20 companies with ratings
│   ├── users.ts                # 60 users with role-based credentials
│   ├── products.ts             # 50 products across 3 categories
│   ├── projects.ts             # 12 projects
│   ├── requisitions.ts         # 15 RFQs with scoring weights
│   ├── contracts.ts            # 42 contracts
│   ├── chats.ts                # 42 chat conversations
│   ├── bids.ts                 # 42 vendor bids
│   ├── comparisons.ts          # 8 L1/L2/L3 comparisons
│   ├── selections.ts           # 6 vendor selections with reasons
│   ├── purchaseOrders.ts       # 5 POs
│   ├── emails.ts               # 55 email logs
│   └── aiData.ts               # Training data + embeddings
└── helpers/
    ├── dateUtils.ts            # Date generation helpers
    ├── priceUtils.ts           # Price/score calculation
    └── idGenerator.ts          # Unique ID helpers
```

---

## 18. Execution Commands

```bash
npm run seed                    # Run full seed
npm run seed -- --clean         # Clean and reseed
npm run seed -- --minimal       # Seed minimal data only
npm run seed -- --skip-chats    # Skip chat data
npm run seed -- --skip-ai       # Skip AI/ML data
```

### Expected Runtime
- Full seed: ~45-90 seconds
- Minimal seed: ~15-20 seconds

---

## Approval Checklist

Please confirm each section:

- [ ] **Companies:** 4 enterprise + 16 vendors with ratings
- [ ] **Users:** Role-based emails/passwords (Admin, Procurement, PM, Manager, Director, VP, Vendor)
- [ ] **Products:** 50 products across IT, Office, Manufacturing
- [ ] **Projects:** 12 projects across companies
- [ ] **Requisitions:** 15 RFQs with configurable scoring weights
- [ ] **L1/L2/L3 System:** Understood as lowest 3 vendor offers
- [ ] **Scoring Formula:** 6 factors with configurable weights
- [ ] **Selection Scenarios:** Clear winner, close competition, trade-offs, edge cases
- [ ] **Approval Renamed:** L1→Manager, L2→Director, L3→VP
- [ ] **Chat Conversations:** 42 chats with all outcomes and depths
- [ ] **Bid Comparisons:** 8 comparisons with full breakdown
- [ ] **Vendor Selections:** All with documented reasons
- [ ] **Purchase Orders:** 5 POs auto-generated from selections
- [ ] **Email Logs:** 55 logs covering all types
- [ ] **AI/ML Data:** Training data + mock embeddings

---

**Once approved, I will proceed with implementation.**
