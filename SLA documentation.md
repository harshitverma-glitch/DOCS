# CRM & Helpdesk SLA Configuration Documentation

## Table of Contents

1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [CRM SLA Configuration](#crm-sla-configuration)
4. [Helpdesk SLA Configuration](#helpdesk-sla-configuration)
5. [Troubleshooting](#troubleshooting)
6. [API Reference](#api-reference)
7. [Best Practices](#best-practices)
8. [Maintenance](#maintenance)

---

## Project Overview

### Purpose

Complete documentation for Service Level Agreement (SLA) configurations in **Frappe CRM v1.52.1** and **Frappe Helpdesk**, providing automated response and resolution time tracking for leads and support tickets.

### Business Context

- **Target Users**: Sales teams, support agents, management
- **Key Stakeholders**: Management requiring SLA compliance tracking
- **Primary Goal**: Automated SLA tracking and enforcement across sales and support operations
- **Secondary Goal**: Performance monitoring and reporting for team accountability

### System Scope

- **CRM SLA**: Dual SLA system for lead response tracking
- **Helpdesk SLA**: Priority-based SLA system for ticket management
- **Automated Status Tracking**: First response and resolution time monitoring
- **SLA Failure Detection**: Automated alerts and status updates

---

## System Architecture

### Technology Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend Layer                           │
├─────────────────────────────────────────────────────────────┤
│  Frappe CRM v1.52.1    │    Frappe Helpdesk               │
├─────────────────────────────────────────────────────────────┤
│                   SLA Management Layer                      │
├─────────────────────────────────────────────────────────────┤
│  CRM SLA Engine        │    Helpdesk SLA Engine           │
│  Server Scripts        │    Priority Management           │
│  Scheduled Jobs        │    Working Hours Calculator      │
├─────────────────────────────────────────────────────────────┤
│                   Data Layer                               │
├─────────────────────────────────────────────────────────────┤
│  CRM Lead              │    HD Ticket                     │
│  Communication         │    HD Service Level Agreement    │
│  CRM SLA               │    HD Ticket Priority            │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. CRM SLA System

```
CRM Lead → Communication Status → SLA Assignment → Response Tracking
    ↓              ↓                    ↓               ↓
  Status      Open/Replied         Response By      Fulfilled/Failed
```

#### 2. Helpdesk SLA System

```
HD Ticket → Priority → SLA Assignment → Response/Resolution Tracking
    ↓          ↓            ↓                    ↓
  Status   High/Med/Low  Response By        Fulfilled/Failed/Paused
                         Resolution By
```

---

## CRM SLA Configuration

### 1. CRM SLA Overview

#### Features

- **Dual SLA System**: Separate SLAs for new leads and follow-up responses
- **Communication-Based Triggers**: SLA based on email direction (inbound/outbound)
- **Automatic Status Updates**: Communication status changes trigger SLA updates
- **24/7 Monitoring**: Continuous SLA failure detection
- **Response Time Tracking**: 2-hour response time standard

### 2. CRM SLA Setup

#### Create Lead Response SLA

1. **Navigate to**: CRM > Settings > CRM Service Level Agreement
2. **Click**: New
3. **Configure**:

| Field | Value | Description |
|-------|-------|-------------|
| **Name** | Lead Response SLA | Unique identifier |
| **DocType** | CRM Lead | Apply to CRM Leads |
| **Condition** | `doc.communication_status == "Open"` | When to apply |
| **Default Priority** | Open | Default priority level |
| **Response Time** | 2 hours | Target response time |
| **Resolution Time** | 2 days | Target resolution time |
| **Start Date** | 2026-01-01 | When SLA becomes active |
| **End Date** | 2027-12-31 | When SLA expires |

#### Create Customer Follow-up Response SLA

1. **Create second SLA**:

| Field | Value | Description |
|-------|-------|-------------|
| **Name** | Customer Follow-up Response SLA | For ongoing conversations |
| **DocType** | CRM Lead | Apply to CRM Leads |
| **Condition** | `doc.communication_status == "Open"` | When to apply |
| **Default Priority** | Open | Default priority level |
| **Response Time** | 2 hours | Target response time |
| **Resolution Time** | 2 days | Target resolution time |

#### SLA Priority Configuration

In the **Priorities** table:

```python
# Priority settings:
Communication Status: Open
Response Time: 2:00:00 (2 hours)
Resolution Time: 2 00:00:00 (2 days)
```

### 3. CRM SLA Automation

#### Communication-Based SLA Triggers

**Outbound Email (Agent Sends):**
- Communication Status → "Replied"
- Lead Status → "Contacted" (if was "New")
- SLA Status → "Fulfilled" (if within deadline)

**Inbound Email (Customer Sends):**
- Communication Status → "Open"
- Lead Status → "OPEN" (if was "Contacted")
- SLA → Reset (new SLA cycle starts)

#### Server Script: Communication Auto-updater

```python
# Name: Auto Update Lead Response Status
# Script Type: Server Script
# DocType: Communication
# Trigger: After Insert

import frappe

def execute(doc):
    if doc.communication_type == 'Communication' and doc.reference_doctype == 'CRM Lead':
        lead_name = doc.reference_name
        
        if not lead_name:
            return
            
        # Get current lead data
        lead = frappe.get_doc('CRM Lead', lead_name)
        current_time = frappe.utils.now_datetime()
        
        if doc.sent_or_received == 'Sent':
            # Outbound email - mark as replied
            frappe.db.set_value('CRM Lead', lead_name, {
                'communication_status': 'Replied',
                'status': 'Contacted' if lead.status == 'New' else lead.status
            }, update_modified=False)
            
            # Mark SLA as fulfilled if within deadline
            if lead.sla and lead.sla_status == 'First Response Due' and lead.response_by:
                if current_time <= lead.response_by:
                    frappe.db.set_value('CRM Lead', lead_name, 'sla_status', 'Fulfilled')
                    
        elif doc.sent_or_received == 'Received':
            # Inbound email - mark as open, reset SLA
            new_status = 'OPEN' if lead.status in ['Contacted', 'OPEN'] else 'New'
            
            frappe.db.set_value('CRM Lead', lead_name, {
                'communication_status': 'Open',
                'status': new_status,
                'sla': '',
                'sla_status': '',
                'response_by': '',
                'resolution_by': ''
            }, update_modified=False)
        
        frappe.db.commit()
```

#### Server Script: SLA Failure Detection

```python
# Name: SLA Failure Detector
# Script Type: Server Script
# DocType: (blank for scheduled)
# Trigger: Cron (*/5 * * * *) - Every 5 minutes

import frappe
from datetime import datetime

def execute():
    # Get all leads with active SLAs
    leads_with_sla = frappe.db.sql("""
        SELECT name, sla, response_by, status, communication_status
        FROM `tabCRM Lead`
        WHERE sla IS NOT NULL 
        AND sla != ''
        AND sla_status = 'First Response Due'
        AND (status = 'New' OR status = 'OPEN')
        AND communication_status = 'Open'
    """, as_dict=True)
    
    current_time = datetime.now()
    
    for lead in leads_with_sla:
        if lead.response_by and lead.response_by < current_time:
            # Mark SLA as Failed
            frappe.db.set_value('CRM Lead', lead.name, 'sla_status', 'Failed')
            frappe.db.commit()
            
            # Optional: Send notification
            frappe.publish_realtime(
                'sla_failed',
                {
                    'lead': lead.name,
                    'message': f'SLA Failed for {lead.name}'
                },
                user='Administrator'
            )
```

### 4. CRM SLA Status Flow

```
New Lead Created
    ↓
[Communication Status: Open]
    ↓
SLA Assigned → [First Response Due]
    ↓
Agent Sends Email → [Replied] → [Fulfilled]
    ↓
Customer Replies → [Open] → New SLA Cycle
    ↓
If No Response by deadline → [Failed]
```

### 5. CRM SLA Fields Reference

**CRM Lead Fields:**

| Field Name | Field Type | Label | Description |
|------------|------------|-------|-------------|
| `sla` | Link | SLA | Links to CRM Service Level Agreement |
| `sla_status` | Select | SLA Status | Fulfilled/Failed/First Response Due |
| `response_by` | Datetime | Response By | Target response time |
| `resolution_by` | Datetime | Resolution By | Target resolution time |
| `communication_status` | Select | Communication Status | Open/Replied |
| `status` | Select | Status | New/Contacted/OPEN/Qualified/etc. |

---

## Helpdesk SLA Configuration

### 1. Helpdesk SLA Overview

#### Features

- **Priority-Based SLAs**: Different response/resolution times per priority
- **Working Hours Support**: 24/7 or business hours configuration
- **Holiday List Integration**: Excludes holidays from SLA calculations
- **SLA Pause/Resume**: Pause SLA when waiting on customer
- **First Response Tracking**: Automatic tracking of agent's first reply
- **Resolution Time Tracking**: Excludes hold time from calculations
- **Agreement Status**: First Response Due, Resolution Due, Fulfilled, Failed, Paused

### 2. Helpdesk SLA Setup

#### Access Helpdesk SLA Settings

**Via Desk:**
```
Helpdesk → Settings → Service Level Agreement
```

**Via Portal (if using Helpdesk Portal):**
```
Settings → SLA → SLA Policies
```

#### Create Default Helpdesk SLA

1. **Navigate to**: HD Service Level Agreement
2. **Click**: New
3. **Configure Basic Settings**:

| Field | Value | Description |
|-------|-------|-------------|
| **Service Level Name** | Standard Support SLA | Unique identifier |
| **Description** | Standard support response times | Optional description |
| **Enabled** | ✓ Checked | Activate the SLA |
| **Default SLA** | ✓ Checked | Apply to all tickets by default |
| **Start Date** | 2026-01-01 | When SLA becomes active |
| **End Date** | 2027-12-31 | When SLA expires (optional) |
| **Apply SLA for Resolution** | ✓ Checked | Track resolution time |

### 3. Priority Configuration

#### Create Priority DocTypes

```python
# Run in bench console
import frappe

priorities = [
    {"name": "High", "color": "Red"},
    {"name": "Medium", "color": "Yellow"},
    {"name": "Low", "color": "Green"}
]

for priority in priorities:
    if not frappe.db.exists("HD Ticket Priority", priority["name"]):
        doc = frappe.get_doc({
            "doctype": "HD Ticket Priority",
            "name": priority["name"],
            "color": priority["color"]
        })
        doc.insert()
        print(f"✅ Created priority: {priority['name']}")
    else:
        print(f"⏭️  Priority already exists: {priority['name']}")

frappe.db.commit()
```

#### Add Priority Levels to SLA

In the **Priorities** table of your SLA:

**Priority 1: High**
```
Priority: High
Default Priority: ✓ (Check as default)
First Response Time: 01:00:00 (1 hour)
Resolution Time: 08:00:00 (8 hours)
```

**Priority 2: Medium**
```
Priority: Medium
Default Priority: □
First Response Time: 02:00:00 (2 hours)
Resolution Time: 24:00:00 (24 hours)
```

**Priority 3: Low**
```
Priority: Low
Default Priority: □
First Response Time: 04:00:00 (4 hours)
Resolution Time: 48:00:00 (48 hours)
```

### 4. Working Hours Configuration

#### Option A: 24/7 Support

In the **Working Hours** table:

```
Monday    - Start: 00:00:00, End: 23:59:59
Tuesday   - Start: 00:00:00, End: 23:59:59
Wednesday - Start: 00:00:00, End: 23:59:59
Thursday  - Start: 00:00:00, End: 23:59:59
Friday    - Start: 00:00:00, End: 23:59:59
Saturday  - Start: 00:00:00, End: 23:59:59
Sunday    - Start: 00:00:00, End: 23:59:59
```

#### Option B: Business Hours (9 AM - 5 PM, Mon-Fri)

```
Monday    - Start: 09:00:00, End: 17:00:00
Tuesday   - Start: 09:00:00, End: 17:00:00
Wednesday - Start: 09:00:00, End: 17:00:00
Thursday  - Start: 09:00:00, End: 17:00:00
Friday    - Start: 09:00:00, End: 17:00:00
```

**Note**: Only add days you want to include. Excluded days are treated as non-working days.

### 5. Holiday List Configuration

#### Create Holiday List

1. **Navigate to**: HD Service Holiday List
2. **Create New**:
   - **Name**: US Holidays 2026
   - **Country**: United States (optional)

3. **Add Holidays**:

```
New Year's Day - 2026-01-01
Memorial Day - 2026-05-25
Independence Day - 2026-07-04
Labor Day - 2026-09-07
Thanksgiving - 2026-11-26
Christmas - 2026-12-25
```

4. **Link to SLA**:
   - In HD Service Level Agreement
   - **Holiday List**: US Holidays 2026

### 6. SLA Status Configuration

#### SLA Fulfilled On

Define which ticket statuses mark the SLA as fulfilled:

**In the "SLA Fulfilled On" table**:

```
Status: Resolved
Status: Closed
```

**Meaning**: When ticket status changes to "Resolved" or "Closed", the SLA is marked as fulfilled and resolution time is calculated.

#### SLA Paused On

Define which statuses pause the SLA timer:

**In the "SLA Paused On" table**:

```
Status: Waiting on Customer
Status: On Hold
```

**Meaning**: When ticket is in these statuses, SLA timer stops. When it moves to another status, timer resumes.

### 7. Assignment Conditions (Advanced)

#### Default SLA (No Condition)

If **Default SLA** is checked, this SLA applies to ALL tickets automatically.

#### Conditional SLA

Uncheck **Default SLA** and add a condition:

**Example 1: VIP Customer SLA**
```python
doc.customer == "VIP Customer" and doc.ticket_type == "Support"
```

**Example 2: Bug Priority SLA**
```python
doc.ticket_type == "Bug" and doc.priority == "High"
```

**Example 3: Team-Specific SLA**
```python
doc.agent_group == "Technical Support Team"
```

### 8. Complete SLA Configuration Script

```python
# Run in bench console to create a complete SLA

import frappe

def create_helpdesk_sla():
    # Check if SLA already exists
    if frappe.db.exists("HD Service Level Agreement", "Standard Support SLA"):
        print("⏭️  SLA already exists")
        return
    
    # Create the SLA document
    sla = frappe.get_doc({
        "doctype": "HD Service Level Agreement",
        "service_level": "Standard Support SLA",
        "description": "Standard support response and resolution times",
        "enabled": 1,
        "default_sla": 1,
        "start_date": "2026-01-01",
        "end_date": "2027-12-31",
        "apply_sla_for_resolution": 1,
        "holiday_list": "US Holidays 2026",  # Create this first
        
        # Priorities
        "priorities": [
            {
                "priority": "High",
                "default_priority": 1,
                "response_time": 3600,  # 1 hour in seconds
                "resolution_time": 28800  # 8 hours in seconds
            },
            {
                "priority": "Medium",
                "default_priority": 0,
                "response_time": 7200,  # 2 hours
                "resolution_time": 86400  # 24 hours
            },
            {
                "priority": "Low",
                "default_priority": 0,
                "response_time": 14400,  # 4 hours
                "resolution_time": 172800  # 48 hours
            }
        ],
        
        # Working Hours (24/7)
        "support_and_resolution": [
            {"workday": "Monday", "start_time": "00:00:00", "end_time": "23:59:59"},
            {"workday": "Tuesday", "start_time": "00:00:00", "end_time": "23:59:59"},
            {"workday": "Wednesday", "start_time": "00:00:00", "end_time": "23:59:59"},
            {"workday": "Thursday", "start_time": "00:00:00", "end_time": "23:59:59"},
            {"workday": "Friday", "start_time": "00:00:00", "end_time": "23:59:59"},
            {"workday": "Saturday", "start_time": "00:00:00", "end_time": "23:59:59"},
            {"workday": "Sunday", "start_time": "00:00:00", "end_time": "23:59:59"}
        ],
        
        # SLA Fulfilled On
        "sla_fulfilled_on": [
            {"status": "Resolved"},
            {"status": "Closed"}
        ],
        
        # SLA Paused On
        "pause_sla_on": [
            {"status": "Waiting on Customer"},
            {"status": "On Hold"}
        ]
    })
    
    sla.insert()
    frappe.db.commit()
    print("✅ Created Standard Support SLA")
    print(f"   Default Priority: High")
    print(f"   Response Times: High=1h, Medium=2h, Low=4h")
    print(f"   Resolution Times: High=8h, Medium=24h, Low=48h")

# Execute
create_helpdesk_sla()
```

### 9. Helpdesk SLA Automation

#### Automatic SLA Application

The SLA is automatically applied when:

1. **New Ticket Created**: 
   - SLA is assigned based on conditions or default SLA
   - `response_by` and `resolution_by` are calculated
   - `agreement_status` is set to "First Response Due"

2. **Priority Changed**:
   - SLA times are recalculated based on new priority
   - `response_by` and `resolution_by` are updated

3. **Status Changed**:
   - If status matches "SLA Fulfilled On" → marks as "Fulfilled"
   - If status matches "SLA Paused On" → marks as "Paused"
   - If status changes from paused → resumes SLA timer

#### SLA Status Transitions

```
New Ticket Created
    ↓
[First Response Due]
    ↓
Agent Replies → [Resolution Due]
    ↓
Status = Resolved/Closed → [Fulfilled]

OR

Response/Resolution missed → [Failed]

OR

Status = On Hold → [Paused] → Status = Open → [Resolution Due]
```

### 10. Helpdesk SLA Fields Reference

**HD Ticket Fields:**

| Field Name | Field Type | Label | Description |
|------------|------------|-------|-------------|
| `sla` | Link | SLA | Links to HD Service Level Agreement |
| `priority` | Link | Priority | Links to HD Ticket Priority |
| `response_by` | Datetime | Response By | Target first response time |
| `resolution_by` | Datetime | Resolution By | Target resolution time |
| `agreement_status` | Select | SLA Status | First Response Due/Resolution Due/Fulfilled/Failed/Paused |
| `first_responded_on` | Datetime | First Responded On | When agent first replied |
| `first_response_time` | Duration | First Response Time | Actual time to first response |
| `resolution_date` | Datetime | Resolution Date | When ticket was resolved |
| `resolution_time` | Duration | Resolution Time | Actual time to resolution |
| `on_hold_since` | Datetime | On Hold Since | When SLA was paused |
| `total_hold_time` | Duration | Total Hold Time | Total time SLA was paused |
| `service_level_agreement_creation` | Datetime | SLA Creation | When SLA was assigned |

### 11. Email Notifications for SLA

#### Create SLA Notification

1. **Navigate to**: HD Notification
2. **Create New**:

**SLA Response Due Notification:**
```
Notification Name: SLA Response Due Soon
Document Type: HD Ticket
Event: Value Change
Condition: doc.agreement_status == "First Response Due"
Send Alert On: Response By - 30 minutes
Recipients: Assigned Agent
Subject: SLA Response Due in 30 Minutes
Message: Ticket {{doc.name}} requires response in 30 minutes
```

**SLA Failed Notification:**
```
Notification Name: SLA Failed Alert
Document Type: HD Ticket
Event: Value Change
Condition: doc.agreement_status == "Failed"
Recipients: Agent + Manager
Subject: SLA Failed for Ticket {{doc.name}}
Message: SLA has been breached for ticket {{doc.name}}
```

---

## Troubleshooting

### CRM SLA Issues

#### 1. SLA Not Attaching to New Leads

**Symptoms**: New leads don't get SLA assigned automatically

**Causes**: 
- SLA validity date expired
- Incorrect condition syntax
- Communication status not "Open"

**Solutions**:

```python
# Check SLA validity
SELECT name, start_date, end_date FROM `tabCRM Service Level Agreement`
WHERE name = 'Lead Response SLA'

# Update end date if expired
UPDATE `tabCRM Service Level Agreement` 
SET end_date = '2027-12-31' 
WHERE name = 'Lead Response SLA'

# Check lead condition
SELECT name, status, communication_status 
FROM `tabCRM Lead` 
WHERE name = 'CRM-LEAD-2026-00001'
```

#### 2. Communication Status Not Updating

**Symptoms**: Emails sent but status remains "Open"

**Causes**:
- Server script not triggered
- Permission issues with frappe.db.set_value()
- Email not linked to CRM Lead

**Solutions**:

```python
# Check communication records
SELECT * FROM `tabCommunication`
WHERE reference_doctype = 'CRM Lead'
AND reference_name = 'CRM-LEAD-2026-00001'
ORDER BY creation DESC

# Test server script manually
lead = frappe.get_doc('CRM Lead', 'CRM-LEAD-2026-00001')
frappe.db.set_value('CRM Lead', 'CRM-LEAD-2026-00001', 'communication_status', 'Replied')
frappe.db.commit()
```

#### 3. SLA Failure Detection Not Running

**Symptoms**: Failed SLAs not being marked as "Failed"

**Causes**:
- Scheduled script not enabled
- Cron job not running
- Script has errors

**Solutions**:

```python
# Check if script exists and is enabled
SELECT name, script_type, disabled FROM `tabServer Script`
WHERE name = 'SLA Failure Detector'

# Run manually to test
import frappe
from datetime import datetime

leads_with_sla = frappe.db.sql("""
    SELECT name, sla, response_by, status, communication_status
    FROM `tabCRM Lead`
    WHERE sla IS NOT NULL 
    AND sla_status = 'First Response Due'
    AND communication_status = 'Open'
""", as_dict=True)

for lead in leads_with_sla:
    if lead.response_by and lead.response_by < datetime.now():
        print(f"Should mark as failed: {lead.name}")
```

### Helpdesk SLA Issues

#### 1. SLA Not Applying to New Tickets

**Symptoms**: New tickets don't have SLA assigned

**Causes**:
- No default SLA configured
- SLA is disabled
- SLA validity dates don't cover current date
- Condition doesn't match ticket

**Solutions**:

```python
# Check if default SLA exists
SELECT name, enabled, default_sla, start_date, end_date 
FROM `tabHD Service Level Agreement`
WHERE enabled = 1

# Check ticket details
SELECT name, priority, status, sla, agreement_status
FROM `tabHD Ticket`
WHERE name = 'HD-TICKET-00001'

# Manually apply SLA
ticket = frappe.get_doc("HD Ticket", "HD-TICKET-00001")
ticket.set_sla()
ticket.save()
```

#### 2. SLA Times Not Calculating Correctly

**Symptoms**: Response By or Resolution By times are incorrect

**Causes**:
- Working hours not configured properly
- Holiday list not set
- Priority not found in SLA
- Time zone issues

**Solutions**:

```python
# Check SLA configuration
sla = frappe.get_doc("HD Service Level Agreement", "Standard Support SLA")
print("Working Days:", sla.get_working_days())
print("Working Hours:", sla.get_working_hours())
print("Holidays:", sla.get_holidays())
print("Priorities:", sla.get_priorities())

# Test SLA calculation
from datetime import datetime
start_time = datetime.now()
priority = "High"
response_by = sla.calc_time(start_time, priority, "response_time")
print(f"Response By: {response_by}")
```

#### 3. SLA Status Not Updating

**Symptoms**: SLA remains "First Response Due" after agent replies

**Causes**:
- `first_responded_on` not being set
- Status not in "SLA Fulfilled On" list
- Server-side validation errors

**Solutions**:

```python
# Check ticket response status
SELECT name, status, first_responded_on, agreement_status, response_by
FROM `tabHD Ticket`
WHERE name = 'HD-TICKET-00001'

# Check SLA fulfilled conditions
SELECT status FROM `tabHD Service Level Agreement Fulfilled On Status`
WHERE parent = 'Standard Support SLA'

# Manually set first response
ticket = frappe.get_doc("HD Ticket", "HD-TICKET-00001")
ticket.first_responded_on = frappe.utils.now_datetime()
ticket.save()
```

#### 4. SLA Pausing Not Working

**Symptoms**: SLA timer doesn't pause when status changes to "On Hold"

**Causes**:
- Status not in "SLA Paused On" list
- `on_hold_since` not being set
- Hold time not being calculated

**Solutions**:

```python
# Check pause conditions
SELECT status FROM `tabHD Pause Service Level Agreement On Status`
WHERE parent = 'Standard Support SLA'

# Check ticket hold status
SELECT name, status, on_hold_since, total_hold_time
FROM `tabHD Ticket`
WHERE name = 'HD-TICKET-00001'

# Manually trigger SLA recalculation
ticket = frappe.get_doc("HD Ticket", "HD-TICKET-00001")
ticket.apply_sla()
ticket.save()
```

#### 5. Priority Not Found in SLA

**Symptoms**: Error "Please add [Priority] priority in [SLA] SLA"

**Causes**:
- Priority exists but not added to SLA priorities table
- Priority name mismatch

**Solutions**:

```python
# Check all priorities
SELECT name FROM `tabHD Ticket Priority`

# Check priorities in SLA
SELECT priority FROM `tabHD Service Level Priority`
WHERE parent = 'Standard Support SLA'

# Add missing priority to SLA
sla = frappe.get_doc("HD Service Level Agreement", "Standard Support SLA")
sla.append("priorities", {
    "priority": "Urgent",
    "response_time": 1800,  # 30 minutes
    "resolution_time": 14400  # 4 hours
})
sla.save()
```

---

## API Reference

### CRM SLA APIs

#### 1. Manual SLA Reset

```python
POST /api/method/frappe.client.set_value

Parameters:
- doctype: "CRM Lead"
- name: lead_name
- fieldname: {"sla": "", "sla_status": "", "response_by": ""}

Response:
{
    "message": "Updated"
}
```

#### 2. Get Lead SLA Status

```python
GET /api/method/frappe.client.get_value

Parameters:
- doctype: "CRM Lead"
- name: lead_name
- fieldname: ["sla", "sla_status", "response_by", "communication_status"]

Response:
{
    "message": {
        "sla": "Lead Response SLA",
        "sla_status": "First Response Due",
        "response_by": "2026-01-12 14:30:00",
        "communication_status": "Open"
    }
}
```

### Helpdesk SLA APIs

#### 1. Get SLA Details

```python
GET /api/method/helpdesk.api.sla.get_sla

Parameters:
- docname (string): SLA name

Response:
{
    "service_level": "Standard Support SLA",
    "priorities": [...],
    "support_and_resolution": [...],
    "sla_fulfilled_on": [...],
    "pause_sla_on": [...]
}
```

#### 2. Duplicate SLA

```python
POST /api/method/helpdesk.api.sla.duplicate_sla

Parameters:
- docname (string): Existing SLA name
- new_name (string): New SLA name

Response:
{
    "name": "New SLA Name",
    "service_level": "New SLA Name"
}
```

#### 3. Manual SLA Reset

```python
POST /api/method/frappe.client.set_value

Parameters:
- doctype: "HD Ticket"
- name: ticket_name
- fieldname: {"sla": "", "agreement_status": "", "response_by": "", "resolution_by": ""}

Response:
{
    "message": "Updated"
}
```

#### 4. Get Ticket SLA Status

```python
GET /api/method/frappe.client.get_value

Parameters:
- doctype: "HD Ticket"
- name: ticket_name
- fieldname: ["sla", "priority", "agreement_status", "response_by", "resolution_by"]

Response:
{
    "message": {
        "sla": "Standard Support SLA",
        "priority": "High",
        "agreement_status": "First Response Due",
        "response_by": "2026-01-12 13:00:00",
        "resolution_by": "2026-01-12 20:00:00"
    }
}
```

---

## Best Practices

### 1. CRM SLA Best Practices

#### Configuration

- **Keep conditions simple**: Complex conditions can cause performance issues
- **Monitor failure rates**: High failure rates indicate unrealistic targets
- **Regular audits**: Review SLA configurations quarterly
- **Communication tracking**: Ensure all emails are properly linked to leads
- **Test automation**: Verify server scripts are triggering correctly

#### Monitoring

```python
# Daily SLA performance check
SELECT 
    DATE(creation) as date,
    COUNT(*) as total_leads,
    SUM(CASE WHEN sla_status = 'Fulfilled' THEN 1 ELSE 0 END) as fulfilled,
    SUM(CASE WHEN sla_status = 'Failed' THEN 1 ELSE 0 END) as failed,
    ROUND(SUM(CASE WHEN sla_status = 'Fulfilled' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as success_rate
FROM `tabCRM Lead`
WHERE creation >= DATE_SUB(NOW(), INTERVAL 30 DAY)
AND sla IS NOT NULL
GROUP BY DATE(creation)
ORDER BY date DESC
```

### 2. Helpdesk SLA Best Practices

#### Configuration

- **Set realistic targets**: Base on historical data and team capacity
- **Use priority appropriately**: Don't mark everything as High priority
- **Configure working hours accurately**: Match actual support availability
- **Update holiday lists**: Keep holiday lists current
- **Monitor hold time**: Excessive hold time may indicate process issues
- **Train agents**: Ensure agents understand SLA importance

#### Priority Guidelines

| Priority | When to Use | Examples |
|----------|-------------|----------|
| **High** | Critical issues, system down, revenue impact | Server outage, payment failure, security breach |
| **Medium** | Standard support requests, non-critical bugs | Feature not working, account questions |
| **Low** | General inquiries, feature requests | How-to questions, documentation requests |

#### Monitoring

```python
# Helpdesk SLA performance report
SELECT 
    DATE(creation) as date,
    priority,
    COUNT(*) as total_tickets,
    SUM(CASE WHEN agreement_status = 'Fulfilled' THEN 1 ELSE 0 END) as fulfilled,
    SUM(CASE WHEN agreement_status = 'Failed' THEN 1 ELSE 0 END) as failed,
    AVG(first_response_time) as avg_response_time,
    AVG(resolution_time) as avg_resolution_time,
    ROUND(SUM(CASE WHEN agreement_status = 'Fulfilled' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as success_rate
FROM `tabHD Ticket`
WHERE creation >= DATE_SUB(NOW(), INTERVAL 30 DAY)
AND sla IS NOT NULL
GROUP BY DATE(creation), priority
ORDER BY date DESC, priority
```

### 3. Performance Optimization

#### Database Indexes

```sql
-- Add indexes for faster SLA queries
ALTER TABLE `tabCRM Lead` ADD INDEX idx_sla_status (sla_status, response_by);
ALTER TABLE `tabCRM Lead` ADD INDEX idx_comm_status (communication_status, status);
ALTER TABLE `tabHD Ticket` ADD INDEX idx_sla_status (agreement_status, response_by);
ALTER TABLE `tabHD Ticket` ADD INDEX idx_resolution (resolution_by, status);
```

#### Scheduled Job Optimization

```python
# Optimize SLA failure detection query
# Instead of loading full documents, use direct SQL updates

def optimized_sla_check():
    # Update failed CRM SLAs
    frappe.db.sql("""
        UPDATE `tabCRM Lead`
        SET sla_status = 'Failed'
        WHERE sla IS NOT NULL
        AND sla_status = 'First Response Due'
        AND communication_status = 'Open'
        AND response_by < NOW()
    """)
    
    # Update failed Helpdesk SLAs
    frappe.db.sql("""
        UPDATE `tabHD Ticket`
        SET agreement_status = 'Failed'
        WHERE sla IS NOT NULL
        AND agreement_status IN ('First Response Due', 'Resolution Due')
        AND (
            (first_responded_on IS NULL AND response_by < NOW())
            OR (resolution_date IS NULL AND resolution_by < NOW())
        )
    """)
    
    frappe.db.commit()
```

---

## Maintenance

### Regular Maintenance Tasks

#### 1. CRM SLA Performance Monitoring

```sql
-- Weekly SLA performance report
SELECT 
    WEEK(creation) as week,
    YEAR(creation) as year,
    COUNT(*) as total_leads,
    SUM(CASE WHEN sla_status = 'Fulfilled' THEN 1 ELSE 0 END) as fulfilled,
    SUM(CASE WHEN sla_status = 'Failed' THEN 1 ELSE 0 END) as failed,
    ROUND(AVG(CASE 
        WHEN sla_status = 'Fulfilled' 
        THEN TIMESTAMPDIFF(MINUTE, creation, modified) 
    END), 2) as avg_response_minutes
FROM `tabCRM Lead`
WHERE creation >= DATE_SUB(NOW(), INTERVAL 90 DAY)
AND sla IS NOT NULL
GROUP BY WEEK(creation), YEAR(creation)
ORDER BY year DESC, week DESC
```

#### 2. Helpdesk SLA Performance Monitoring

```sql
-- Monthly SLA performance by team
SELECT 
    MONTH(t.creation) as month,
    YEAR(t.creation) as year,
    t.agent_group as team,
    t.priority,
    COUNT(*) as total_tickets,
    SUM(CASE WHEN t.agreement_status = 'Fulfilled' THEN 1 ELSE 0 END) as fulfilled,
    SUM(CASE WHEN t.agreement_status = 'Failed' THEN 1 ELSE 0 END) as failed,
    ROUND(AVG(t.first_response_time), 2) as avg_response_seconds,
    ROUND(AVG(t.resolution_time), 2) as avg_resolution_seconds
FROM `tabHD Ticket` t
WHERE t.creation >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
AND t.sla IS NOT NULL
GROUP BY MONTH(t.creation), YEAR(t.creation), t.agent_group, t.priority
ORDER BY year DESC, month DESC, team, priority
```

#### 3. System Health Check

```python
# Schedule daily health check
def daily_sla_health_check():
    issues = []
    
    # Check for failed CRM SLAs
    failed_crm_slas = frappe.db.count('CRM Lead', {
        'sla_status': 'Failed',
        'creation': ('>=', frappe.utils.add_days(frappe.utils.today(), -1))
    })
    
    if failed_crm_slas > 10:
        issues.append(f"High CRM SLA failure rate: {failed_crm_slas} failures today")
    
    # Check for failed Helpdesk SLAs
    failed_hd_slas = frappe.db.count('HD Ticket', {
        'agreement_status': 'Failed',
        'creation': ('>=', frappe.utils.add_days(frappe.utils.today(), -1))
    })
    
    if failed_hd_slas > 15:
        issues.append(f"High Helpdesk SLA failure rate: {failed_hd_slas} failures today")
    
    # Check for SLAs without response_by
    missing_response_by = frappe.db.sql("""
        SELECT COUNT(*) as count FROM `tabHD Ticket`
        WHERE sla IS NOT NULL
        AND response_by IS NULL
        AND creation >= DATE_SUB(NOW(), INTERVAL 1 DAY)
    """)[0][0]
    
    if missing_response_by > 0:
        issues.append(f"Tickets with SLA but no response_by: {missing_response_by}")
    
    # Send alerts if issues found
    if issues:
        frappe.sendmail(
            recipients=['admin@company.com'],
            subject='SLA System Health Alert',
            message='Issues detected:\n' + '\n'.join(issues)
        )
```

### Backup & Recovery

#### Export SLA Configuration

```python
# Export all SLA configurations
def export_sla_config():
    import json
    
    crm_slas = frappe.get_all('CRM Service Level Agreement', 
                         fields=['*'])
    
    hd_slas = []
    for sla_name in frappe.get_all('HD Service Level Agreement', pluck='name'):
        sla = frappe.get_doc('HD Service Level Agreement', sla_name)
        hd_slas.append(sla.as_dict())
    
    server_scripts = frappe.get_all('Server Script',
                                   filters={'disabled': 0, 'script_type': ['in', ['DocType Event', 'Scheduler Event']]},
                                   fields=['*'])
    
    config = {
        'crm_slas': crm_slas,
        'hd_slas': hd_slas,
        'scripts': server_scripts,
        'export_date': frappe.utils.now()
    }
    
    # Save to file
    with open('/tmp/sla_config_backup.json', 'w') as f:
        json.dump(config, f, indent=2, default=str)
    
    print("✅ SLA configuration exported to /tmp/sla_config_backup.json")
    return config
```

#### Import SLA Configuration

```python
# Import SLA configuration to new site
def import_sla_config(config_file_path):
    import json
    
    with open(config_file_path, 'r') as f:
        config = json.load(f)
    
    # Import CRM SLAs
    for sla in config['crm_slas']:
        if not frappe.db.exists('CRM Service Level Agreement', sla['name']):
            sla_doc = frappe.get_doc({
                'doctype': 'CRM Service Level Agreement',
                **sla
            })
            sla_doc.insert()
            print(f"✅ Imported CRM SLA: {sla['name']}")
    
    # Import Helpdesk SLAs
    for sla in config['hd_slas']:
        if not frappe.db.exists('HD Service Level Agreement', sla['service_level']):
            sla_doc = frappe.get_doc({
                'doctype': 'HD Service Level Agreement',
                **sla
            })
            sla_doc.insert()
            print(f"✅ Imported Helpdesk SLA: {sla['service_level']}")
    
    # Import server scripts
    for script in config['scripts']:
        if not frappe.db.exists('Server Script', script['name']):
            script_doc = frappe.get_doc({
                'doctype': 'Server Script',
                **script
            })
            script_doc.insert()
            print(f"✅ Imported Server Script: {script['name']}")
    
    frappe.db.commit()
    print("✅ SLA configuration import complete")
```

---

## Appendix

### A. Quick Reference Commands

```bash
# Bench commands
bench --site mysite.local migrate
bench --site mysite.local clear-cache
bench --site mysite.local console

# Database queries
bench --site mysite.local mariadb

# Restart services
bench restart

# View logs
bench --site mysite.local watch
```

### B. Common Field Names Reference

**CRM Lead:**
- `sla` - CRM Service Level Agreement link
- `sla_status` - Fulfilled/Failed/First Response Due
- `response_by` - Target response datetime
- `resolution_by` - Target resolution datetime
- `communication_status` - Open/Replied

**HD Ticket:**
- `sla` - HD Service Level Agreement link
- `agreement_status` - SLA status
- `priority` - HD Ticket Priority link
- `response_by` - Target first response datetime
- `resolution_by` - Target resolution datetime
- `first_responded_on` - Actual first response datetime
- `first_response_time` - Time taken for first response
- `resolution_date` - Actual resolution datetime
- `resolution_time` - Time taken for resolution
- `on_hold_since` - When SLA was paused
- `total_hold_time` - Total time SLA was paused

### C. Time Format Reference

**Duration Format:**
- Format: `HH:MM:SS`
- Examples:
  - 1 hour: `01:00:00`
  - 2 hours: `02:00:00`
  - 30 minutes: `00:30:00`
  - 8 hours: `08:00:00`
  - 24 hours: `24:00:00`
  - 48 hours: `48:00:00`

**Seconds Conversion:**
- 30 minutes: 1800 seconds
- 1 hour: 3600 seconds
- 2 hours: 7200 seconds
- 4 hours: 14400 seconds
- 8 hours: 28800 seconds
- 24 hours: 86400 seconds
- 48 hours: 172800 seconds

---

## Version History

### v1.0.0 (Current)

**CRM SLA:**
- ✅ Dual SLA system implementation
- ✅ Communication-based status automation
- ✅ 5-minute failure detection
- ✅ 2-hour response time standard

**Helpdesk SLA:**
- ✅ Priority-based SLA system
- ✅ Working hours and holiday support
- ✅ SLA pause/resume functionality
- ✅ First response and resolution tracking
- ✅ Agreement status automation

### Planned Features (v1.1.0)

- 🔄 Advanced reporting dashboard (CRM + Helpdesk)
- 🔄 SLA escalation rules
- 🔄 Multi-tier SLA support
- 🔄 Customer satisfaction tracking
- 🔄 Agent performance metrics
- 🔄 Predictive SLA analytics

---

## Support & Documentation

### Key Resources

- [Frappe Framework Documentation](https://frappeframework.com/docs)
- [Frappe CRM Documentation](https://docs.frappe.io/crm)
- [Frappe Helpdesk Documentation](https://docs.frappe.io/helpdesk)

### Contact Information

- **Developer**: Harshit Verma
- **Document Version**: 1.0
- **Last Updated**: January 12, 2026
- **Status**: Complete - CRM & Helpdesk SLA Only

---

*This documentation covers only the Service Level Agreement (SLA) configurations for CRM and Helpdesk systems. For specific implementation questions, refer to the troubleshooting section or contact the development team.*

