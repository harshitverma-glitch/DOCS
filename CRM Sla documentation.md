# Frappe CRM & ERPNext Integration - Complete Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [Core Features](#core-features)
4. [Installation & Setup](#installation--setup)
5. [Configuration Guide](#configuration-guide)
6. [Implementation Details](#implementation-details)
7. [Troubleshooting](#troubleshooting)
8. [API Reference](#api-reference)
9. [Best Practices](#best-practices)
10. [Maintenance](#maintenance)

---

## Project Overview

### Purpose
A comprehensive customer relationship and order management system that integrates **Frappe CRM v1.52.1** with **ERPNext v15.84.0**, providing unified visibility across the entire customer lifecycle from lead generation to production completion.

### Business Context
- **Target Users**: Production managers, support teams, operations teams
- **Key Stakeholders**: Management requiring SLA compliance tracking
- **Primary Goal**: Complete customer interaction and order history visibility within CRM
- **Secondary Goal**: Real-time production pipeline status tracking

### System Scope
- Automated order imports from Shopify and Etsy
- Dual SLA system with 2-hour response times
- Real-time production status tracking (OPS, Material, Production, QA)
- Inter-company order relationships through complex reference chains
- HD Ticket order history integration

---

## System Architecture

### Technology Stack
```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend Layer                           │
├─────────────────────────────────────────────────────────────┤
│  Frappe CRM v1.52.1    │    ERPNext v15.84.0              │
├─────────────────────────────────────────────────────────────┤
│                   Integration Layer                         │
├─────────────────────────────────────────────────────────────┤
│  Custom App: crm_order_extension                           │
│  Server Scripts: SLA Management, Order Sync                │
│  Client Scripts: UI Enhancements, Manual Sync             │
├─────────────────────────────────────────────────────────────┤
│                   Data Layer                               │
├─────────────────────────────────────────────────────────────┤
│  Shopify Connector  │  Custom Etsy App  │  ERPNext DB     │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. CRM Extension App Structure
```
crm_order_extension/
├── crm_order_extension/
│   ├── __init__.py
│   ├── hooks.py
│   ├── config/
│   │   └── docs.py
│   └── crm_order_extension/
│       ├── __init__.py
│       ├── api/
│       │   └── crm_lead_orders.py
│       ├── doctype/
│       └── public/
│           └── js/
│               ├── crm_lead.js
│               └── crm_deal.js
├── requirements.txt
├── setup.py
└── license.txt
```

#### 2. Data Relationships
```
Customer → Sales Orders → Purchase Orders → Work Orders
    ↓           ↓              ↓              ↓
CRM Lead → Order Items → Production → Status Updates
```

---

## Core Features

### 1. Dual SLA System

#### Lead Response SLA
- **Trigger Condition**: `doc.status == "New"` AND `doc.communication_status == "Open"`
- **Response Time**: 2 hours
- **Resolution Time**: 2 days
- **Operating Hours**: 24/7
- **Failure Detection**: Every 5 minutes via scheduled script

#### Customer Follow-up Response SLA
- **Trigger Condition**: `doc.status == "OPEN"` AND `doc.communication_status == "Open"`
- **Response Time**: 2 hours
- **Resolution Time**: 2 days
- **Operating Hours**: 24/7
- **Auto-transition**: From Lead Response SLA when status changes to "OPEN"

### 2. Order History Tracking

#### Real-time Status Display
- **OPS Status**: APPROVED/PENDING/BLOCKED
- **Material Status**: IN_STOCK/OUT_OF_STOCK/ORDERED
- **Production Status**: DONE/IN_PROGRESS/PENDING
- **QA Status**: AWAITING/APPROVED/REJECTED

#### Color-coded Status Badges
```css
Green:  Approved/Complete states
Blue:   In-progress states  
Yellow: Pending/Review states
Red:    Blocked/Error states
```

### 3. Automated Communication Management

#### Email Direction Detection
- **Outbound Emails**: Status → "Contacted"/"Replied", SLA → "Fulfilled"
- **Inbound Emails**: Status → "OPEN", New SLA cycle starts
- **Auto-status Updates**: Via Communication DocType server scripts

### 4. Production Workflow Integration

#### Complex Relationship Chain
```
Customer Sales Order → Inter-company Purchase Order → Work Order
       ↓                        ↓                      ↓
   Order Details            PO Items              Custom Status Fields
```

#### Status Field Mapping
- `custom_ops_status` → OPS Status
- `custom_ingredients_status` → Material Status  
- `custom_production_status` → Production Status
- `custom_qa_status` → QA Status

---

## Installation & Setup

### Prerequisites
- ERPNext v15.84.0 installed and configured
- Frappe CRM v1.52.1 installed
- Shopify Connector configured (optional)
- Custom Etsy integration app (optional)

### 1. Custom App Installation

#### Install via GitHub
```bash
# Navigate to frappe-bench
cd frappe-bench

# Install the custom app
bench get-app https://github.com/your-username/crm_order_extension.git

# Install on site
bench --site your-site install-app crm_order_extension

# Migrate
bench --site your-site migrate
```

#### Manual Installation
```bash
# Create app locally
bench new-app crm_order_extension

# Install on site
bench --site your-site install-app crm_order_extension
```

### 2. Required DocTypes

#### CRM Lead Order Item (Child Table)
```python
# Fields required:
- sales_order (Link: Sales Order)
- item_code (Data)
- item_name (Data)
- qty (Float)
- rate (Currency)
- amount (Currency)
- delivery_date (Date)
- ops_status (Select: APPROVED/PENDING/BLOCKED)
- material_status (Select: IN_STOCK/OUT_OF_STOCK/ORDERED)
- production_status (Select: DONE/IN_PROGRESS/PENDING)
- qa_status (Select: AWAITING/APPROVED/REJECTED)
```

#### HD Ticket Order Item (Child Table)
```python
# Fields required:
- sales_order (Link: Sales Order)
- item_code (Data)
- item_name (Data)
- qty (Float)
- rate (Currency)
- amount (Currency)
- ops_status (Select)
- material_status (Select)
- production_status (Select)
- qa_status (Select)
```

---

## Configuration Guide

### 1. SLA Configuration

#### Create Lead Response SLA
1. Navigate to **CRM > Settings > CRM Service Level Agreement**
2. Create new SLA:
   - **Name**: Lead Response SLA
   - **DocType**: CRM Lead
   - **Condition**: `doc.communication_status == "Open"`
   - **Default Priority**: Open
   - **Response Time**: 2 hours
   - **Resolution Time**: 2 days

#### Create Customer Follow-up Response SLA
1. Create second SLA:
   - **Name**: Customer Follow-up Response SLA
   - **DocType**: CRM Lead
   - **Condition**: `doc.communication_status == "Open"`
   - **Default Priority**: Open
   - **Response Time**: 2 hours
   - **Resolution Time**: 2 days

#### SLA Priority Configuration
```python
# Priority table settings:
Communication Status: Open
Response Time: 2:00:00
Resolution Time: 2 00:00:00
```

### 2. Form Customization

#### CRM Lead Form
1. **Customize Form** → CRM Lead
2. Add **Order Details** tab
3. Add table field:
   - **Fieldname**: `custom_order_history`
   - **Label**: Order History
   - **Type**: Table
   - **Options**: CRM Lead Order Item

#### HD Ticket Form
1. **Customize Form** → HD Ticket
2. Add **Order Details** tab
3. Add table field:
   - **Fieldname**: `custom_order_history`
   - **Label**: Order History
   - **Type**: Table
   - **Options**: HD Ticket Order Item

---

## Implementation Details

### 1. Server Scripts

#### SLA Failure Detection (Scheduled Script)
```python
# Name: SLA Failure Detector
# Script Type: Server Script
# DocType: (blank for scheduled)
# Trigger: Cron (*/5 * * * *)

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

#### Communication Auto-updater
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

#### Order History Sync API
```python
# Name: CRM Lead Fetch Orders
# Script Type: API
# Method: erpnext_integrations.erpnext_integrations.api.fetch_orders

import frappe
import json

@frappe.whitelist()
def fetch_orders(lead_name):
    try:
        # Get lead data
        lead = frappe.get_doc('CRM Lead', lead_name)
        if not lead.lead_name:
            return {'success': False, 'message': 'Lead name not found'}
        
        # Find matching customer (case-insensitive)
        customers = frappe.db.sql("""
            SELECT name FROM `tabCustomer`
            WHERE LOWER(customer_name) = %s
        """, (lead.lead_name.lower(),), as_dict=True)
        
        if not customers:
            return {'success': False, 'message': 'No customer found'}
        
        customer_name = customers[0].name
        
        # Get sales orders
        sales_orders = frappe.db.sql("""
            SELECT name, transaction_date, net_total, delivery_date
            FROM `tabSales Order`
            WHERE customer = %s AND docstatus = 1
            ORDER BY transaction_date DESC
        """, (customer_name,), as_dict=True)
        
        # Clear existing order history
        frappe.db.sql("""
            DELETE FROM `tabCRM Lead Order Item`
            WHERE parent = %s
        """, (lead_name,))
        
        # Process each sales order
        for so in sales_orders:
            # Get sales order items
            so_items = frappe.db.sql("""
                SELECT item_code, item_name, qty, rate
                FROM `tabSales Order Item`
                WHERE parent = %s
            """, (so.name,), as_dict=True)
            
            for item in so_items:
                # Get work order status
                work_orders = frappe.db.sql("""
                    SELECT 
                        custom_ops_status,
                        custom_ingredients_status,
                        custom_production_status,
                        custom_qa_status
                    FROM `tabWork Order`
                    WHERE sales_order = %s
                    AND production_item = %s
                    ORDER BY creation DESC
                    LIMIT 1
                """, (so.name, item.item_code), as_dict=True)
                
                # Set status values
                if work_orders:
                    wo = work_orders[0]
                    ops_status = wo.custom_ops_status or 'PENDING'
                    material_status = wo.custom_ingredients_status or 'PENDING'
                    production_status = wo.custom_production_status or 'PENDING'
                    qa_status = wo.custom_qa_status or 'AWAITING'
                else:
                    # Default values when no work order
                    ops_status = 'PENDING'
                    material_status = 'PENDING'
                    production_status = 'PENDING'
                    qa_status = 'AWAITING'
                
                # Create order item record
                order_item = frappe.get_doc({
                    'doctype': 'CRM Lead Order Item',
                    'parent': lead_name,
                    'parenttype': 'CRM Lead',
                    'parentfield': 'custom_order_history',
                    'sales_order': so.name,
                    'item_code': item.item_code,
                    'item_name': item.item_name,
                    'qty': item.qty,
                    'rate': item.rate,
                    'amount': so.net_total,
                    'delivery_date': so.delivery_date,
                    'ops_status': ops_status,
                    'material_status': material_status,
                    'production_status': production_status,
                    'qa_status': qa_status
                })
                order_item.insert()
        
        frappe.db.commit()
        return {'success': True, 'message': f'Synced {len(sales_orders)} orders'}
        
    except Exception as e:
        frappe.log_error(f"Error in fetch_orders: {str(e)}")
        return {'success': False, 'message': str(e)}
```

### 2. Client Scripts

#### CRM Lead Order History Button
```javascript
// Name: CRM Lead Order History
// DocType: CRM Lead

frappe.ui.form.on('CRM Lead', {
    onload: function(frm) {
        // Add custom button
        frm.add_custom_button(__('Fetch Order History'), function() {
            fetch_order_history(frm);
        }, __('Actions'));
    }
});

function fetch_order_history(frm) {
    // Show loading indicator
    frappe.show_alert({
        message: __('Fetching order history...'),
        indicator: 'blue'
    });
    
    // Call API
    frappe.call({
        method: 'erpnext_integrations.erpnext_integrations.api.fetch_orders',
        args: {
            lead_name: frm.doc.name
        },
        callback: function(response) {
            if (response.message.success) {
                frappe.show_alert({
                    message: response.message.message,
                    indicator: 'green'
                });
                frm.reload_doc();
            } else {
                frappe.show_alert({
                    message: response.message.message,
                    indicator: 'red'
                });
            }
        }
    });
}

// Color-coded status badges
frappe.ui.form.on('CRM Lead', {
    refresh: function(frm) {
        add_status_colors(frm);
    }
});

function add_status_colors(frm) {
    setTimeout(() => {
        // OPS Status colors
        $('.form-grid [data-fieldname="ops_status"]').each(function() {
            const value = $(this).find('input').val();
            if (value === 'APPROVED') $(this).addClass('status-green');
            else if (value === 'PENDING') $(this).addClass('status-yellow');
            else if (value === 'BLOCKED') $(this).addClass('status-red');
        });
        
        // Material Status colors
        $('.form-grid [data-fieldname="material_status"]').each(function() {
            const value = $(this).find('input').val();
            if (value === 'IN_STOCK') $(this).addClass('status-green');
            else if (value === 'ORDERED') $(this).addClass('status-yellow');
            else if (value === 'OUT_OF_STOCK') $(this).addClass('status-red');
        });
        
        // Production Status colors
        $('.form-grid [data-fieldname="production_status"]').each(function() {
            const value = $(this).find('input').val();
            if (value === 'DONE') $(this).addClass('status-green');
            else if (value === 'IN_PROGRESS') $(this).addClass('status-blue');
            else if (value === 'PENDING') $(this).addClass('status-yellow');
        });
        
        // QA Status colors
        $('.form-grid [data-fieldname="qa_status"]').each(function() {
            const value = $(this).find('input').val();
            if (value === 'APPROVED') $(this).addClass('status-green');
            else if (value === 'AWAITING') $(this).addClass('status-yellow');
            else if (value === 'REJECTED') $(this).addClass('status-red');
        });
    }, 500);
}

// CSS for status colors (add to custom CSS)
/*
.status-green { background-color: #d4edda !important; }
.status-blue { background-color: #cce5f0 !important; }
.status-yellow { background-color: #fff3cd !important; }
.status-red { background-color: #f8d7da !important; }
*/
```

#### HD Ticket Order History
```javascript
// Name: HD Ticket Order History
// DocType: HD Ticket

frappe.ui.form.on('HD Ticket', {
    onload: function(frm) {
        frm.add_custom_button(__('Fetch Order History'), function() {
            fetch_hd_ticket_orders(frm);
        }, __('Actions'));
    }
});

function fetch_hd_ticket_orders(frm) {
    if (!frm.doc.contact) {
        frappe.msgprint(__('Contact is required to fetch order history'));
        return;
    }
    
    frappe.show_alert({
        message: __('Fetching order history...'),
        indicator: 'blue'
    });
    
    frappe.call({
        method: 'erpnext_integrations.erpnext_integrations.api.hd_ticket_fetch_orders',
        args: {
            contact: frm.doc.contact,
            ticket_name: frm.doc.name
        },
        callback: function(response) {
            if (response.message && response.message.success) {
                frappe.show_alert({
                    message: response.message.message,
                    indicator: 'green'
                });
                frm.reload_doc();
            } else {
                frappe.show_alert({
                    message: response.message ? response.message.message : 'Failed to fetch orders',
                    indicator: 'red'
                });
            }
        }
    });
}
```

---

## Troubleshooting

### Common Issues

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
SET end_date = '2026-12-31' 
WHERE name = 'Lead Response SLA'

# Check lead condition
SELECT name, status, communication_status 
FROM `tabCRM Lead` 
WHERE name = 'CRM-LEAD-2026-00001'
```

#### 2. Order History Not Syncing
**Symptoms**: "Fetch Order History" returns no data
**Causes**:
- Case-sensitive customer name matching
- Missing Work Order relationships
- Incorrect sales order field mapping

**Solutions**:
```sql
-- Debug customer matching
SELECT customer_name FROM `tabCustomer` 
WHERE LOWER(customer_name) LIKE '%kelly%'

-- Check sales order relationships
SELECT so.name, so.customer, wo.name as work_order
FROM `tabSales Order` so
LEFT JOIN `tabWork Order` wo ON wo.sales_order = so.name
WHERE so.customer = 'Kelly'

-- Verify work order status fields
SELECT name, custom_ops_status, custom_ingredients_status, 
       custom_production_status, custom_qa_status
FROM `tabWork Order`
WHERE sales_order = 'SAL-ORD-2025-02478'
```

#### 3. Communication Status Not Updating
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
AND reference_name = 'CRM-LEAD-2025-00734'
ORDER BY creation DESC

# Test server script manually
lead = frappe.get_doc('CRM Lead', 'CRM-LEAD-2025-00734')
frappe.db.set_value('CRM Lead', 'CRM-LEAD-2025-00734', 'communication_status', 'Replied')
frappe.db.commit()
```

### Performance Issues

#### 1. Slow Order History Loading
**Problem**: Large customers with many orders cause timeouts
**Solution**: Implement pagination and caching
```python
# Add limit to sales order query
sales_orders = frappe.db.sql("""
    SELECT name, transaction_date, net_total, delivery_date
    FROM `tabSales Order`
    WHERE customer = %s AND docstatus = 1
    ORDER BY transaction_date DESC
    LIMIT 50
""", (customer_name,), as_dict=True)
```

#### 2. SLA Checker Performance
**Problem**: 5-minute SLA checker slowing down system
**Solution**: Add database indexes
```sql
-- Add indexes for faster SLA queries
ALTER TABLE `tabCRM Lead` ADD INDEX idx_sla_status (sla_status, response_by);
ALTER TABLE `tabCRM Lead` ADD INDEX idx_comm_status (communication_status, status);
```

---

## API Reference

### Core API Methods

#### 1. Fetch Lead Order History
```python
POST /api/method/erpnext_integrations.erpnext_integrations.api.fetch_orders

Parameters:
- lead_name (string): CRM Lead name

Response:
{
    "success": true,
    "message": "Synced 5 orders"
}
```

#### 2. HD Ticket Order History
```python
POST /api/method/erpnext_integrations.erpnext_integrations.api.hd_ticket_fetch_orders

Parameters:
- contact (string): Contact name
- ticket_name (string): HD Ticket name

Response:
{
    "success": true,
    "message": "Found customer: John Smith. Synced 3 orders"
}
```

#### 3. Manual SLA Reset
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

### Webhook Integration

#### Shopify Order Webhook
```python
# URL: https://yoursite.com/api/method/erpnext_shopify.shopify_integration.order_webhook
# Triggers: Sales Order creation → Order history sync
```

#### Etsy Order Webhook  
```python
# URL: https://yoursite.com/api/method/custom_etsy_app.webhook.order_created
# Triggers: Sales Order creation → Order history sync
```

---

## Best Practices

### 1. Development Guidelines

#### Server Script Best Practices
- Use `frappe.db.set_value()` for single field updates
- Avoid manual `frappe.db.commit()` in transaction contexts
- Use direct SQL for complex queries with proper parameterization
- Implement error handling with `frappe.log_error()`

```python
# Good practice
try:
    frappe.db.set_value('CRM Lead', lead_name, 'status', 'Contacted')
    frappe.db.commit()
except Exception as e:
    frappe.log_error(f"Error updating lead status: {str(e)}")
```

#### Client Script Best Practices
- Use setTimeout for DOM manipulation after form load
- Implement loading indicators for long-running operations
- Add proper error handling and user feedback

```javascript
// Good practice
frappe.show_alert({
    message: __('Processing...'),
    indicator: 'blue'
});
```

### 2. Data Management

#### Customer Matching Strategy
- Always use case-insensitive matching for customer names
- Use LIKE queries for partial matches when appropriate
- Validate customer existence before processing orders

```sql
-- Best practice for customer matching
SELECT name FROM `tabCustomer`
WHERE LOWER(TRIM(customer_name)) = LOWER(TRIM(%s))
```

#### Work Order Status Mapping
- Use consistent status field naming conventions
- Implement fallback values for missing status fields
- Map status values to user-friendly display text

```python
status_map = {
    'APPROVED': 'Approved',
    'PENDING': 'Pending',
    'BLOCKED': 'Blocked',
    'IN_PROGRESS': 'In Progress'
}
```

### 3. Performance Optimization

#### Database Queries
- Use appropriate indexes on frequently queried fields
- Limit result sets with LIMIT clauses
- Use EXISTS instead of COUNT when checking for record existence

#### Caching Strategy
- Cache frequently accessed customer-lead mappings
- Implement result caching for complex work order status queries
- Use Redis for session-based temporary storage

### 4. Error Handling

#### Server Script Error Patterns
```python
def safe_execute(func, *args, **kwargs):
    try:
        return func(*args, **kwargs)
    except frappe.ValidationError as e:
        frappe.log_error(f"Validation Error: {str(e)}")
        return {'success': False, 'message': str(e)}
    except Exception as e:
        frappe.log_error(f"Unexpected Error: {str(e)}")
        return {'success': False, 'message': 'An unexpected error occurred'}
```

#### Client Script Error Patterns
```javascript
function safe_api_call(method, args, callback) {
    frappe.call({
        method: method,
        args: args,
        callback: callback,
        error: function(response) {
            frappe.show_alert({
                message: __('An error occurred. Please try again.'),
                indicator: 'red'
            });
            console.error('API Error:', response);
        }
    });
}
```

---

## Maintenance

### Regular Maintenance Tasks

#### 1. SLA Performance Monitoring
```sql
-- Check SLA response times
SELECT 
    DATE(creation) as date,
    COUNT(*) as total_leads,
    SUM(CASE WHEN sla_status = 'Fulfilled' THEN 1 ELSE 0 END) as fulfilled,
    SUM(CASE WHEN sla_status = 'Failed' THEN 1 ELSE 0 END) as failed
FROM `tabCRM Lead`
WHERE creation >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY DATE(creation)
ORDER BY date DESC
```

#### 2. Order Sync Health Check
```sql
-- Check for leads with missing order history
SELECT l.name, l.lead_name, COUNT(o.name) as order_count
FROM `tabCRM Lead` l
LEFT JOIN `tabCRM Lead Order Item` o ON o.parent = l.name
WHERE EXISTS (
    SELECT 1 FROM `tabCustomer` c 
    WHERE LOWER(c.customer_name) = LOWER(l.lead_name)
)
GROUP BY l.name, l.lead_name
HAVING order_count = 0
```

#### 3. Data Cleanup Scripts

#### Remove Duplicate Order Items
```python
def cleanup_duplicate_orders():
    duplicates = frappe.db.sql("""
        SELECT parent, sales_order, item_code, COUNT(*) as count
        FROM `tabCRM Lead Order Item`
        GROUP BY parent, sales_order, item_code
        HAVING count > 1
    """, as_dict=True)
    
    for dup in duplicates:
        # Keep only the latest record
        records = frappe.db.sql("""
            SELECT name FROM `tabCRM Lead Order Item`
            WHERE parent = %s AND sales_order = %s AND item_code = %s
            ORDER BY creation DESC
        """, (dup.parent, dup.sales_order, dup.item_code))
        
        # Delete older duplicates
        for record in records[1:]:
            frappe.delete_doc('CRM Lead Order Item', record[0])
```

### 4. Monitoring & Alerts

#### Setup System Health Monitoring
```python
# Schedule daily health check
def daily_health_check():
    issues = []
    
    # Check for failed SLAs
    failed_slas = frappe.db.count('CRM Lead', {
        'sla_status': 'Failed',
        'creation': ('>=', frappe.utils.add_days(frappe.utils.today(), -1))
    })
    
    if failed_slas > 10:
        issues.append(f"High SLA failure rate: {failed_slas} failures today")
    
    # Check for stale orders
    stale_orders = frappe.db.sql("""
        SELECT COUNT(*) as count
        FROM `tabSales Order`
        WHERE docstatus = 1
        AND creation >= DATE_SUB(NOW(), INTERVAL 7 DAY)
        AND NOT EXISTS (
            SELECT 1 FROM `tabCRM Lead Order Item` o
            WHERE o.sales_order = `tabSales Order`.name
        )
    """)[0][0]
    
    if stale_orders > 5:
        issues.append(f"Orders not synced to CRM: {stale_orders}")
    
    # Send alerts if issues found
    if issues:
        frappe.sendmail(
            recipients=['admin@company.com'],
            subject='CRM Integration Health Alert',
            message='Issues detected:\n' + '\n'.join(issues)
        )
```

### Backup & Recovery

#### Export Configuration
```python
# Export SLA configurations
def export_sla_config():
    slas = frappe.get_all('CRM Service Level Agreement', 
                         fields=['name', 'condition', 'response_time'])
    
    server_scripts = frappe.get_all('Server Script',
                                   filters={'disabled': 0},
                                   fields=['name', 'script_type', 'script'])
    
    config = {
        'slas': slas,
        'scripts': server_scripts,
        'export_date': frappe.utils.now()
    }
    
    return config
```

#### Import Configuration
```python
# Import configuration to new site
def import_sla_config(config_data):
    # Create SLAs
    for sla in config_data['slas']:
        if not frappe.db.exists('CRM Service Level Agreement', sla['name']):
            sla_doc = frappe.get_doc({
                'doctype': 'CRM Service Level Agreement',
                'name': sla['name'],
                'condition': sla['condition'],
                'response_time': sla['response_time']
            })
            sla_doc.insert()
    
    # Create server scripts
    for script in config_data['scripts']:
        if not frappe.db.exists('Server Script', script['name']):
            script_doc = frappe.get_doc({
                'doctype': 'Server Script',
                'name': script['name'],
                'script_type': script['script_type'],
                'script': script['script']
            })
            script_doc.insert()
```

---

## Version History

### v1.0.0 (Current)
- ✅ Dual SLA system implementation
- ✅ Order history tracking for CRM Leads
- ✅ HD Ticket integration
- ✅ Communication status automation
- ✅ Custom Frappe app architecture
- ✅ Production status real-time sync

### Planned Features (v1.1.0)
- 🔄 Deal-specific SLA handling for converted leads
- 🔄 Advanced reporting dashboard
- 🔄 Bulk order history synchronization
- 🔄 Mobile-optimized status displays
- 🔄 Advanced notification system

---

## Support & Documentation

### Key Resources
- [Frappe Framework Documentation](https://frappeframework.com/docs)
- [ERPNext Documentation](https://docs.erpnext.com/)
- [Frappe CRM Documentation](https://docs.frappe.io/crm)

### Contact Information
- **Developer**: Harshit Verma
- **Project Repository**: [GitHub Link]
- **Issue Tracking**: [Project Issues]

### Contributing
1. Fork the repository
2. Create feature branch (`git checkout -b feature/new-feature`)
3. Commit changes (`git commit -am 'Add new feature'`)
4. Push to branch (`git push origin feature/new-feature`)
5. Create Pull Request

---

*This documentation covers the complete integration system developed over multiple months of iterative development and testing. For specific implementation questions, refer to the troubleshooting section or contact the development team.*