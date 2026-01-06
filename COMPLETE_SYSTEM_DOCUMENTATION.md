# Complete System Documentation: CRM, Helpdesk & RingCentral Integration

**Document Version:** 1.0  
**Last Updated:** January 2, 2026  
**System:** Frappe CRM & Helpdesk with ERPNext Integration

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [CRM Module Enhancements](#crm-module-enhancements)
3. [Helpdesk Module Enhancements](#helpdesk-module-enhancements)
4. [Order History Integration](#order-history-integration)
5. [Email Transfer System](#email-transfer-system)
6. [RingCentral Integration](#ringcentral-integration)
7. [Email Routing System (Server Script)](#email-routing-system-server-script)
8. [Deployment Guide](#deployment-guide)
9. [API Reference](#api-reference)
10. [Troubleshooting](#troubleshooting)

---

## Executive Summary

This document provides comprehensive technical documentation for all enhancements made to the CRM and Helpdesk modules, including RingCentral telephony integration, email transfer capabilities, and order history tracking from ERPNext.

### Key Statistics

- **Total Features Implemented:** 15
- **CRM Features:** 5
- **Helpdesk Features:** 9
- **Email Routing:** 1 (Server Script)
- **API Endpoints Created:** 20+
- **Frontend Components:** 15+

### Technology Stack

**Frontend:**
- Vue 3 with Composition API
- frappe-ui component library
- Vite build system
- TypeScript (Helpdesk) / JavaScript (CRM)

**Backend:**
- Python 3.12
- Frappe Framework
- ERPNext
- RingCentral SDK

---

## CRM Module Enhancements

### 1. Sales Order Validation on Deals

**Purpose:** Enforce business rule requiring at least one Sales Order before marking a Deal as "Won".

**Files Modified:**
- `apps/crm/crm/fcrm/doctype/crm_deal/crm_deal.json`
- `apps/crm/crm/fcrm/doctype/crm_deal/crm_deal.py`

**Files Created:**
- `apps/crm/crm/fcrm/doctype/crm_deal_sales_orders/crm_deal_sales_orders.json`
- `apps/crm/crm/fcrm/doctype/crm_deal_sales_orders/crm_deal_sales_orders.py`
- `apps/crm/crm/fcrm/doctype/crm_deal_sales_orders/__init__.py`

**Implementation:**

Created a child DocType to store Sales Order references:

```python
class CRMDealSalesOrders(Document):
    # Fields: sales_order (Link to Sales Order)
    pass
```

Added validation method in CRM Deal:

```python
def validate_sales_orders_on_won(self):
    """Validate that at least one Sales Order is provided when status is Won."""
    if self.status and frappe.get_cached_value("CRM Deal Status", self.status, "type") == "Won":
        if not self.sales_orders or len(self.sales_orders) == 0:
            frappe.throw(
                _("Please add at least one Sales Order before marking the deal as Won."),
                frappe.ValidationError
            )
```

**User Experience:**
1. New "Sales Orders" tab appears in Deal form
2. Users can add multiple Sales Orders via table
3. Validation triggers when changing status to "Won"
4. Error message displayed if no Sales Orders added

**Testing:**
- ✅ Create Deal without Sales Orders → Try to mark as Won → Validation error
- ✅ Add Sales Order → Mark as Won → Success
- ✅ Multiple Sales Orders can be added
- ✅ Sales Orders persist after save

---

### 2. Lead Auto-Merge by Phone Number

**Purpose:** Automatically consolidate call logs when a mobile number is added to an existing lead that matches another lead's phone number.

**Files Modified:**
- `apps/crm/crm/fcrm/doctype/crm_lead/crm_lead.py`

**Implementation:**

Phone number change detection in validate():

```python
def validate(self):
    # ... existing code ...
    
    # Handle mobile number updates - merge call logs from duplicate leads
    if not self.is_new() and self.has_value_changed("mobile_no") and self.mobile_no:
        self.handle_mobile_number_update()
```

Phone normalization and duplicate finding:

```python
def handle_mobile_number_update(self):
    """Handle mobile number field update - merge call logs from duplicates"""
    if not self.mobile_no:
        return
    
    from crm.integrations.ringcentral_utils import normalize_phone_number
    normalized_phone = normalize_phone_number(self.mobile_no)
    
    if not normalized_phone:
        return
    
    frappe.logger().info(f"Mobile number updated for lead {self.name} to {self.mobile_no}")
    self.merge_call_logs_by_phone(normalized_phone)
```

Call log merging:

```python
def merge_call_logs_by_phone(self, phone_number):
    """Find and merge call logs from other leads with same phone number"""
    # Find duplicate leads
    duplicate_leads = frappe.db.sql("""
        SELECT name, lead_name, mobile_no
        FROM `tabCRM Lead`
        WHERE mobile_no = %s AND name != %s
    """, (phone_number, self.name), as_dict=True)
    
    for dup_lead in duplicate_leads:
        # Find call logs via reference_docname
        call_logs = frappe.db.get_all(
            "CRM Call Log",
            filters={
                "reference_doctype": "CRM Lead",
                "reference_docname": dup_lead.name
            },
            pluck="name"
        )
        
        # Also find via Dynamic Link table
        linked_call_logs = frappe.db.sql("""
            SELECT parent FROM `tabDynamic Link`
            WHERE link_doctype = 'CRM Lead'
            AND link_name = %s
            AND parenttype = 'CRM Call Log'
        """, (dup_lead.name,), as_dict=True)
        
        # Re-link call logs
        all_call_logs = set(call_logs)
        all_call_logs.update([log.parent for log in linked_call_logs])
        
        for log_name in all_call_logs:
            call_log = frappe.get_doc("CRM Call Log", log_name)
            if call_log.reference_docname == dup_lead.name:
                call_log.db_set("reference_docname", self.name, update_modified=False)
            
            for link in call_log.links:
                if link.link_name == dup_lead.name:
                    frappe.db.set_value("Dynamic Link", link.name, "link_name", self.name)
        
        frappe.db.commit()
```

**Phone Normalization:**
- Removes: spaces, dashes, parentheses, dots
- `+1 (555) 123-4567` → `15551234567`
- `555.123.4567` → `5551234567`

**Workflow:**
```
Old Lead (No Phone) → Customer Calls → New Lead Created (With Phone + Call Logs)
                                              ↓
                              User Adds Phone to Old Lead
                                              ↓
                              System Detects Duplicate Phone
                                              ↓
                              Call Logs Transferred to Old Lead
```

---

### 3. Call Transcription Viewing

**Purpose:** Allow users to view call transcripts fetched from RingCentral API with database caching.

**Files Created:**
- `apps/crm/crm/api/call_transcription.py`

**Files Modified:**
- `apps/crm/frontend/src/components/CallArea.vue`

**Backend API:**

```python
@frappe.whitelist()
def get_or_create_transcript(call_log_name):
    """Get transcript from cache or fetch from RingCentral"""
    call_log = frappe.get_doc("CRM Call Log", call_log_name)
    
    # Check cache
    if call_log.transcript:
        return {
            "success": True,
            "transcript": call_log.transcript,
            "cached": True
        }
    
    # Fetch from RingCentral
    from crm.integrations.ringcentral_client import RingCentralClient
    client = RingCentralClient()
    
    if not client.authenticate_auto():
        return {"success": False, "message": "Authentication failed"}
    
    recording_id = extract_recording_id(call_log.recording_url)
    transcript_text = client.get_ringcentral_transcript(recording_id)
    
    # Cache in database
    call_log.db_set("transcript", transcript_text, update_modified=False)
    
    return {
        "success": True,
        "transcript": transcript_text,
        "cached": False
    }

@frappe.whitelist()
def refresh_transcript(call_log_name):
    """Force refresh transcript from RingCentral"""
    # Similar to above but always fetches fresh
```

**Features:**
- ✅ Database caching for performance
- ✅ Refresh button to re-fetch
- ✅ Copy to clipboard functionality
- ✅ Loading states and error handling
- ✅ Cached indicator

---

### 4. Email Transfer System (CRM)

**Purpose:** Transfer email communications between CRM Leads, Deals, Contacts, and Helpdesk Tickets.

**Files Created:**
- `apps/crm/crm/api/email_transfer.py`

**Files Modified:**
- `apps/crm/frontend/src/components/EmailTransferDialog.vue`
- `apps/crm/frontend/src/utils/emailTransfer.js`

**Key Functions:**

**Get Communications:**
```python
@frappe.whitelist()
def get_communications_for_transfer(doctype, name):
    """Get list of Communication records for transfer"""
    communications = []
    comm_names = frappe.get_all(
        "Communication",
        filters={
            "reference_doctype": doctype,
            "reference_name": name,
            "communication_medium": "Email"
        },
        fields=["name"],
        order_by="creation asc"
    )
    
    for comm_name in comm_names:
        comm = frappe.get_doc("Communication", comm_name.name)
        communications.append({
            "name": comm.name,
            "subject": comm.subject,
            "sender": comm.sender,
            "recipients": comm.recipients,
            "creation": comm.creation,
            "content": comm.content,
            "text_content": comm.text_content
        })
    
    return communications
```

**Copy Communication:**
```python
def copy_communication(comm_name, target_doctype, target_name):
    """Duplicate communication with new reference"""
    original_comm = frappe.get_doc("Communication", comm_name)
    
    new_comm = frappe.new_doc("Communication")
    
    # Copy all fields explicitly
    new_comm.communication_type = original_comm.communication_type
    new_comm.subject = original_comm.subject
    new_comm.content = original_comm.content  # CRITICAL
    new_comm.text_content = original_comm.text_content  # CRITICAL
    new_comm.sender = original_comm.sender
    new_comm.recipients = original_comm.recipients
    new_comm.message_id = original_comm.message_id
    new_comm.in_reply_to = original_comm.in_reply_to
    
    # Set new reference
    new_comm.reference_doctype = target_doctype
    new_comm.reference_name = str(target_name)
    new_comm.status = "Linked"
    
    new_comm.insert(ignore_permissions=True)
    return new_comm.name
```

**Transfer to Helpdesk:**
```python
@frappe.whitelist()
def transfer_to_helpdesk(lead_name, communication_ids=None, delete_source=True):
    """Transfer CRM Lead to HD Ticket with selected communications"""
    lead = frappe.get_doc("CRM Lead", lead_name)
    
    # Extract customer info
    customer_name = f"{lead.first_name} {lead.last_name}".strip()
    customer_email = lead.email or lead.email_id
    customer_phone = lead.mobile_no or lead.phone
    
    # Create or update Contact
    contact = create_or_update_contact(customer_email, customer_name, customer_phone)
    
    # Create HD Ticket
    ticket = frappe.new_doc("HD Ticket")
    ticket.subject = get_latest_email_subject(communication_ids)
    ticket.raised_by = customer_email
    ticket.contact = contact.name
    ticket.contact_number = customer_phone
    ticket.insert(ignore_permissions=True)
    frappe.db.commit()
    
    # Copy selected communications
    transferred_count = 0
    for comm_id in communication_ids:
        copy_communication(comm_id, "HD Ticket", ticket.name)
        transferred_count += 1
        frappe.db.commit()
    
    # Delete source if requested
    if delete_source:
        frappe.delete_doc("CRM Lead", lead_name, force=True)
    
    return {
        "success": True,
        "ticket_name": ticket.name,
        "transferred_count": transferred_count
    }
```

**Features:**
- ✅ Selective email transfer (choose which emails)
- ✅ Maintains email thread integrity
- ✅ Preserves email content and attachments
- ✅ Creates/updates Contacts automatically
- ✅ Optional source deletion
- ✅ Transfer notes generation

---

### 5. Graceful Payload Processing

**Purpose:** Handle cases where RingCentral payload documents are deleted before processing completes.

**Files Modified:**
- `apps/erpnext/erpnext/crm/doctype/ringcentral_crm_payload/ringcentral_crm_payload.py`

**Implementation:**

```python
def process_crm_payload(payload_name):
    """Process RingCentral webhook payload"""
    # Check if document exists first
    if not frappe.db.exists("RingCentral CRM Payload", payload_name):
        frappe.logger().warning(f"Payload {payload_name} not found, may have been deleted")
        return
    
    try:
        payload = frappe.get_doc("RingCentral CRM Payload", payload_name)
        # Process payload...
    except Exception as e:
        error_message = str(e)
        
        # Update error status only if document still exists
        try:
            if frappe.db.exists("RingCentral CRM Payload", payload_name):
                payload = frappe.get_doc("RingCentral CRM Payload", payload_name)
                payload.db_set('processing_status', 'Error', update_modified=False)
                payload.db_set('error_message', error_message[:500], update_modified=False)
                frappe.db.commit()
        except Exception as update_error:
            frappe.logger().error(f"Failed to update error status: {str(update_error)}")
```

**Benefits:**
- ✅ No crashes when payloads deleted
- ✅ Graceful error handling
- ✅ Background jobs don't get stuck
- ✅ Better error logging

---

## Helpdesk Module Enhancements

### 1. Click-to-Call Functionality

**Purpose:** Enable agents to initiate calls directly from ticket interface using RingCentral.

**Files Created:**
- `apps/helpdesk/helpdesk/api/ringcentral_calling.py`
- `apps/helpdesk/desk/src/components/RingCentralCallUI.vue`

**Files Modified:**
- `apps/helpdesk/desk/src/components/ticket/TicketAgentContact.vue`
- `apps/helpdesk/desk/src/components/layouts/DesktopLayout.vue`
- `apps/helpdesk/desk/src/components/layouts/MobileLayout.vue`

**Backend API:**

```python
@frappe.whitelist()
def create_manual_call_log(phone_number, contact_name=None, ticket_name=None):
    """Create call log when agent initiates outbound call"""
    call_log = frappe.new_doc("Helpdesk Call Log")
    call_log.type = "Outgoing"
    call_log.from_number = get_company_phone()
    call_log.to_number = phone_number
    call_log.status = "Initiated"
    
    if ticket_name:
        call_log.ticket = ticket_name
    
    if contact_name:
        call_log.contact = contact_name
    
    call_log.insert(ignore_permissions=True)
    return call_log.name

@frappe.whitelist()
def get_contact_by_phone(phone_number):
    """Find existing contact by phone number"""
    # Search logic...
```

**Frontend Component:**

The RingCentralCallUI.vue component provides:
- Draggable popup UI
- Contact information display
- Call initiation via desktop app or web
- Automatic call log creation

**Call Protocols:**
- Desktop: `rcapp://r/call?number={phone}`
- Web: `https://app.ringcentral.com/calls/call?number={phone}`

**Workflow:**
1. Agent clicks call button in ticket contact card
2. Popup appears with contact info
3. Agent clicks "Call" button
4. System tries desktop app first (`rcapp://`)
5. Falls back to web URL after 1 second
6. Call log created automatically
7. Call log linked to ticket and contact

---

### 2. Call Recording Playback

**Purpose:** Allow agents to play call recordings directly in the ticket interface.

**Files Created:**
- `apps/helpdesk/helpdesk/api/recording.py`
- `apps/helpdesk/desk/src/components/CallRecordingPlayer.vue`

**Files Modified:**
- `apps/helpdesk/desk/src/components/ticket/TicketCallLogs.vue`

**Backend API:**

```python
@frappe.whitelist()
def fetch_and_save_helpdesk_recording(call_log_name: str):
    """Fetch recording from RingCentral and save to ERPNext"""
    call_log = frappe.get_doc("Helpdesk Call Log", call_log_name)
    
    if not call_log.recording_url:
        return {"success": False, "message": "No recording URL"}
    
    # Authenticate with RingCentral
    from crm.integrations.ringcentral_client import RingCentralClient
    client = RingCentralClient()
    
    if not client.authenticate_auto():
        return {"success": False, "message": "Authentication failed"}
    
    # Download recording
    recording_id = extract_recording_id(call_log.recording_url)
    audio_content = client.download_recording(recording_id)
    
    # Save to ERPNext files
    file_doc = frappe.get_doc({
        "doctype": "File",
        "file_name": f"recording_{call_log.name}.mp3",
        "content": audio_content,
        "is_private": 1,
        "attached_to_doctype": "Helpdesk Call Log",
        "attached_to_name": call_log.name
    })
    file_doc.insert()
    
    # Update call log
    call_log.db_set("recording_file", file_doc.file_url)
    
    return {
        "success": True,
        "file_url": file_doc.file_url
    }
```

**Features:**
- ✅ HTML5 audio player
- ✅ Play/pause controls
- ✅ Progress bar with seek
- ✅ Time display (current/total)
- ✅ Download option
- ✅ Loading states
- ✅ Error handling with retry

---

### 3. Call Transcription Viewing

**Purpose:** Display call transcripts from RingCentral in Helpdesk tickets.

**Files Created:**
- `apps/helpdesk/helpdesk/api/transcript.py`
- `apps/helpdesk/desk/src/components/CallTranscriptModal.vue`

**Files Modified:**
- `apps/helpdesk/desk/src/components/ticket/TicketCallLogs.vue`
- `apps/helpdesk/helpdesk/integrations/ringcentral_utils.py`

**Backend API:**

```python
@frappe.whitelist()
def get_or_create_transcript(call_log_name):
    """Get transcript from cache or fetch from RingCentral"""
    call_log = frappe.get_doc("Helpdesk Call Log", call_log_name)
    
    # Check cache
    if call_log.transcript:
        return {
            "success": True,
            "transcript": call_log.transcript,
            "cached": True
        }
    
    # Fetch from RingCentral
    from crm.integrations.ringcentral_client import RingCentralClient
    client = RingCentralClient()
    
    if not client.authenticate_auto():
        return {"success": False, "message": "Authentication failed"}
    
    recording_id = extract_recording_id(call_log.ringcentral_recording_url)
    transcript_text = client.get_ringcentral_transcript(recording_id)
    
    # Cache in database
    call_log.db_set("transcript", transcript_text, update_modified=False)
    
    return {
        "success": True,
        "transcript": transcript_text,
        "cached": False
    }
```

**Features:**
- ✅ Database caching for performance
- ✅ Refresh button to re-fetch
- ✅ Copy to clipboard functionality
- ✅ Loading states and error handling
- ✅ Cached indicator (purple badge)
- ✅ Modal dialog UI

---

### 4. Automatic Ticket Phone Merging

**Purpose:** Automatically transfer call logs when phone number is added to a ticket.

**Files Modified:**
- `apps/helpdesk/helpdesk/helpdesk/doctype/hd_ticket/hd_ticket.py`

**Files Created:**
- `apps/helpdesk/helpdesk/api/ticket_phone_utils.py`

**Automatic Merging:**

```python
def handle_phone_number_update(self):
    """Handle contact_mobile field update - merge call logs from duplicates"""
    if not self.has_value_changed("contact_mobile") or not self.contact_mobile:
        return
    
    from helpdesk.integrations.ringcentral_utils import normalize_phone_number
    normalized_phone = normalize_phone_number(self.contact_mobile)
    
    if not normalized_phone:
        return
    
    # Find duplicate tickets
    duplicate_tickets = frappe.db.sql("""
        SELECT name, subject, contact_mobile
        FROM `tabHD Ticket`
        WHERE contact_mobile = %s AND name != %s
    """, (normalized_phone, self.name), as_dict=True)
    
    for dup_ticket in duplicate_tickets:
        # Find call logs
        call_logs = frappe.db.get_all(
            "Helpdesk Call Log",
            filters={"ticket": dup_ticket.name},
            pluck="name"
        )
        
        # Transfer to current ticket
        for log_name in call_logs:
            frappe.db.set_value("Helpdesk Call Log", log_name, "ticket", self.name)
        
        frappe.db.commit()
```

**Manual Utilities:**

```python
@frappe.whitelist()
def merge_call_logs_by_phone(ticket_name):
    """Manually trigger call log merging for a ticket"""
    # Implementation...

@frappe.whitelist()
def find_duplicate_tickets_by_phone(phone_number):
    """Find all tickets with a specific phone number"""
    # Implementation...

@frappe.whitelist()
def bulk_merge_duplicate_tickets():
    """Process all duplicate tickets and merge to oldest"""
    # Implementation...
```

**Workflow:**
```
Email Ticket (No Phone) → Customer Calls → Call Ticket Created (With Phone)
                                                      ↓
                                      Agent Adds Phone to Email Ticket
                                                      ↓
                                          System Detects Duplicate Phone
                                                      ↓
                                          Call Logs Transfer to Email Ticket
```

---

### 5. Phone Number Save Fix

**Purpose:** Properly save phone numbers with validation and hooks using standard Frappe API.

**Files Modified:**
- `apps/helpdesk/desk/src/components/ticket/TicketAgentDetails.vue`

**Implementation:**

```javascript
const saveMobileNo = async () => {
  if (!mobileNo.value) {
    showToast({
      title: 'Error',
      text: 'Phone number cannot be empty',
      icon: 'x',
      iconClasses: 'text-red-600'
    })
    return
  }
  
  try {
    // Use standard Frappe API to trigger validation hooks
    await call('frappe.client.set_value', {
      doctype: 'HD Ticket',
      name: ticket.value.name,
      fieldname: 'contact_mobile',
      value: mobileNo.value
    })
    
    // Reload ticket to get updated data
    await ticket.value.reload()
    
    showToast({
      title: 'Success',
      text: 'Phone number saved successfully',
      icon: 'check',
      iconClasses: 'text-green-600'
    })
    
    isEditingMobile.value = false
  } catch (error) {
    showToast({
      title: 'Error',
      text: error.message || 'Failed to save phone number',
      icon: 'x',
      iconClasses: 'text-red-600'
    })
  }
}
```

**Benefits:**
- ✅ Triggers validation hooks properly
- ✅ Proper error handling
- ✅ Toast notifications for user feedback
- ✅ Automatic ticket reload
- ✅ Works with auto-merge feature

---

### 6. Recording URL Extraction Fix

**Purpose:** Extract recording URLs from Pabbly webhook payloads and construct proper RingCentral API URLs.

**Files Modified:**
- `apps/helpdesk/helpdesk/api/ringcentral_webhook.py`
- `apps/helpdesk/helpdesk/integrations/ringcentral_utils.py`

**Implementation:**

```python
def process_pabbly_webhook(payload):
    """Process Pabbly webhook for call data"""
    # Extract recording data
    recording_data = payload.get('recording', {})
    recording_id = recording_data.get('id')
    
    if recording_id:
        # Construct full RingCentral API URL
        recording_url = construct_recording_url(recording_id)
        
        # Update call log
        call_log = find_call_log(payload)
        if call_log:
            call_log.db_set('recording_url', recording_url)
            call_log.db_set('ringcentral_recording_url', recording_url)

def construct_recording_url(recording_id, account_id="~"):
    """Construct full RingCentral recording URL"""
    base_url = "https://platform.ringcentral.com"
    return f"{base_url}/restapi/v1.0/account/{account_id}/recording/{recording_id}/content"
```

**Features:**
- ✅ Parses Pabbly webhook payloads
- ✅ Extracts recording IDs
- ✅ Constructs proper API URLs
- ✅ Updates call logs automatically
- ✅ Error handling for missing data

---

### 7. Contact Mobile Field Fix

**Purpose:** Ensure Contact is created before Ticket to enable proper field fetching via `fetch_from`.

**Files Modified:**
- `apps/helpdesk/helpdesk/integrations/ringcentral_utils.py`
- `apps/helpdesk/desk/src/components/ticket/TicketAgentDetails.vue`

**Implementation:**

```python
def create_ticket_from_call(phone_number, caller_name):
    """Create ticket with proper Contact creation sequence"""
    # Step 1: Create/update Contact FIRST
    contact = frappe.get_doc({
        "doctype": "Contact",
        "first_name": caller_name,
        "mobile_no": phone_number
    })
    contact.insert()
    frappe.db.commit()  # CRITICAL: Commit Contact before Ticket
    
    # Step 2: Create Ticket with fetch_from working
    ticket = frappe.new_doc("HD Ticket")
    ticket.contact = contact.name
    # contact_mobile will be fetched from Contact automatically
    ticket.insert()
    
    return ticket
```

**Benefits:**
- ✅ Proper field fetching via `fetch_from`
- ✅ No race conditions
- ✅ Validation works correctly
- ✅ Editable mobile field in UI

---

### 8. Order History Display

**Purpose:** Show customer's ERPNext Sales Orders in Helpdesk tickets with production status tracking.

**Files Created:**
- `apps/helpdesk/helpdesk/api/order_history.py`
- `apps/helpdesk/desk/src/components/ticket/TicketOrderDetails.vue`

**Backend API:**

```python
@frappe.whitelist()
def fetch_ticket_order_history(ticket_name, customer_name):
    """Fetch order history for HD Ticket from Sales Orders"""
    # Find customers matching name
    customers = frappe.db.sql("""
        SELECT name, customer_name
        FROM `tabCustomer`
        WHERE LOWER(TRIM(customer_name)) = LOWER(%s)
    """, customer_name, as_dict=True)
    
    if not customers:
        return {"success": False, "message": "No customers found"}
    
    # Get Sales Orders
    all_sales_orders = []
    for customer in customers:
        sales_orders = frappe.db.sql("""
            SELECT name, customer, transaction_date, delivery_status,
                   delivery_date, status, net_total, shopify_order_number
            FROM `tabSales Order`
            WHERE customer = %s AND docstatus != 2
            ORDER BY transaction_date DESC
        """, customer.name, as_dict=True)
        all_sales_orders.extend(sales_orders)
    
    # Get ticket document
    ticket_doc = frappe.get_doc("HD Ticket", ticket_name)
    
    # Process each Sales Order
    total_items_added = 0
    for so in all_sales_orders:
        # Get Work Orders for production status
        work_orders = get_work_orders_for_sales_order(so.name)
        aggregate_status = _calculate_aggregate_production_status(work_orders)
        
        # Get items
        so_items = frappe.db.sql("""
            SELECT name, item_code, item_name, qty, rate
            FROM `tabSales Order Item`
            WHERE parent = %s
        """, so.name, as_dict=True)
        
        for item in so_items:
            # Get detailed statuses
            ops_status, mat_status, pro_status, qa_status = _get_work_order_statuses(
                so.name, item.item_code, item.get('name')
            )
            
            # Add to ticket order history
            ticket_doc.append("custom_order_history", {
                "sales_order": so.name,
                "order_date": so.transaction_date,
                "item_code": item.item_code,
                "item_name": item.item_name,
                "qty": item.qty,
                "rate": item.rate,
                "amount": so.net_total,
                "delivery_status": so.delivery_status or "Not Delivered",
                "delivery_date": so.delivery_date,
                "order_status": so.status,
                "ops_status": ops_status,
                "custom_ingredients_status": mat_status,
                "pro_status": pro_status,
                "qa_status": qa_status,
                "aggregate_production_status": aggregate_status
            })
            total_items_added += 1
    
    if total_items_added > 0:
        ticket_doc.save()
        frappe.db.commit()
    
    return {
        "success": True,
        "message": f"Successfully synced {total_items_added} items from {len(all_sales_orders)} orders"
    }
```

**Production Status Tracking:**

The system tracks multiple status fields:
- **OPS Status**: Work order operational status
- **Ingredients Status**: Material availability
- **Production Status**: Manufacturing progress
- **QA Status**: Quality assurance state
- **Aggregate Status**: Overall production state

**Status Priority:**
1. CANCELLED (highest priority)
2. BLOCKED FACTORY
3. BLOCKED OPS
4. QA REWORK
5. NEW → WIP → DONE (progression)

---

### 9. Email Transfer (Helpdesk)

**Purpose:** Transfer emails between Helpdesk tickets and CRM entities.

**Files Created:**
- `apps/helpdesk/helpdesk/api/email_transfer.py`

**Files Modified:**
- `apps/helpdesk/desk/src/components/EmailTransferDialog.vue`
- `apps/helpdesk/desk/src/utils/emailTransfer.js`

**Transfer to CRM:**

```python
@frappe.whitelist()
def transfer_to_crm(ticket_name, communication_ids=None, delete_source=True):
    """Transfer HD Ticket to CRM Lead with selected communications"""
    ticket = frappe.get_doc("HD Ticket", ticket_name)
    
    # Get customer info
    customer_name = ""
    if ticket.contact:
        contact = frappe.get_doc("Contact", ticket.contact)
        customer_name = f"{contact.first_name} {contact.last_name}".strip()
    
    customer_email = ticket.raised_by
    customer_phone = ticket.contact_number
    
    # Parse name
    first_name, last_name = parse_name_into_first_last(customer_name)
    
    # Create CRM Lead
    lead = frappe.new_doc("CRM Lead")
    lead.first_name = first_name
    lead.last_name = last_name
    lead.email = customer_email
    lead.mobile_no = customer_phone
    lead.status = "New"
    lead.source = "Support Transfer"
    lead.insert(ignore_permissions=True)
    frappe.db.commit()
    
    # Copy communications
    transferred_count = 0
    for comm_id in communication_ids:
        copy_communication(comm_id, "CRM Lead", lead.name)
        transferred_count += 1
        frappe.db.commit()
    
    # Delete source if requested
    if delete_source:
        frappe.delete_doc("HD Ticket", ticket_name, force=True)
    
    return {
        "success": True,
        "lead_name": lead.name,
        "transferred_count": transferred_count
    }
```

**Features:**
- ✅ Bidirectional transfer (Ticket ↔ Lead)
- ✅ Contact creation/update
- ✅ Email thread preservation
- ✅ Selective transfer (choose emails)
- ✅ Source deletion option
- ✅ Transfer notes generation

---

## Order History Integration

### Overview

The Order History feature integrates ERPNext Sales Orders with CRM Leads, Deals, and Helpdesk Tickets, providing agents with complete visibility into customer purchase history and production status.

### Architecture

```
ERPNext Sales Order
    ↓
Purchase Order (Inter-company)
    ↓
Inter Company Sales Order
    ↓
Work Order
    ↓
Production Status → CRM/Helpdesk
```

### Customer Matching Logic

**Exact Name Match:**
```python
if ' ' in customer_name:
    # Full name: exact match
    customers = frappe.db.sql("""
        SELECT name, customer_name
        FROM `tabCustomer`
        WHERE LOWER(TRIM(customer_name)) = LOWER(%s)
    """, customer_name, as_dict=True)
else:
    # First name only: match customers with no last name
    customers = frappe.db.sql("""
        SELECT name, customer_name
        FROM `tabCustomer`
        WHERE LOWER(TRIM(customer_name)) = LOWER(%s)
        AND customer_name NOT LIKE '% %'
    """, customer_name, as_dict=True)
```

### Work Order Status Tracking

**Status Fields:**
- **OPS Status**: Work order operational status
- **Ingredients Status**: Material availability (custom_ingredients_status)
- **Production Status**: Manufacturing progress (custom_production_status)
- **QA Status**: Quality assurance state (custom_qa_status)
- **Aggregate Status**: Overall production state

**Aggregate Status Calculation:**

```python
def _calculate_aggregate_production_status(work_orders):
    """Calculate aggregate production status from multiple Work Orders"""
    if not work_orders:
        return 'N/A'
    
    # Priority mapping (lower number = higher priority)
    PRIORITY = {
        'CANCELLED': 1,
        'BLOCKED FACTORY': 2,
        'BLOCKED_FACTORY': 2,
        'BLOCKED OPS': 3,
        'BLOCKED_OPS': 3,
        'QA REWORK': 4
    }
    
    statuses = [wo.get('custom_production_status') for wo in work_orders]
    
    # Check priority statuses first
    for status in statuses:
        if status.upper() in PRIORITY:
            return status
    
    # Check progression
    upper_statuses = [s.upper() for s in statuses]
    
    if all(s in ['DONE', 'COMPLETED'] for s in upper_statuses):
        return 'DONE'
    
    if 'NEW' in upper_statuses:
        return 'NEW'
    
    if any(s in ['WIP', 'IN PROGRESS'] for s in upper_statuses):
        return 'WIP'
    
    return statuses[0] if statuses else 'N/A'
```

### API Endpoints

**CRM Lead:**
```python
frappe.call(
    method='crm.api.order_history.fetch_lead_order_history',
    args={
        'lead_name': 'LEAD-2025-00001',
        'customer_name': 'John Doe'
    }
)
```

**CRM Deal:**
```python
frappe.call(
    method='crm.api.order_history.fetch_deal_order_history',
    args={
        'deal_name': 'DEAL-2025-00001',
        'customer_name': 'John Doe'
    }
)
```

**Helpdesk Ticket:**
```python
frappe.call(
    method='helpdesk.api.order_history.fetch_ticket_order_history',
    args={
        'ticket_name': 'HD-TICKET-00001',
        'customer_name': 'John Doe'
    }
)
```

---

## Email Transfer System

### Overview

The Email Transfer System enables seamless movement of email communications between CRM entities (Leads, Deals, Contacts) and Helpdesk Tickets, maintaining thread integrity and customer context.

### Communication Structure

```
Communication Document
    ├── reference_doctype (HD Ticket / CRM Lead)
    ├── reference_name (document ID)
    ├── content (HTML email body)
    ├── text_content (plain text)
    ├── sender
    ├── recipients
    ├── message_id (for threading)
    └── in_reply_to (for threading)
```

### Transfer Workflows

**CRM Lead → Helpdesk Ticket:**
1. User selects emails to transfer
2. System extracts customer info (email, phone, name)
3. Creates/updates Contact document
4. Creates new HD Ticket
5. Copies selected Communications
6. Optionally deletes source Lead

**Helpdesk Ticket → CRM Lead:**
1. User selects emails to transfer
2. System extracts customer info from Ticket
3. Creates new CRM Lead
4. Copies selected Communications
5. Optionally deletes source Ticket

### Email Thread Integrity

**Threading Maintained:**
- `message_id`: Unique identifier for each email
- `in_reply_to`: Links replies to original emails
- Thread order preserved by creation timestamp

### Customer Email Extraction

**Internal Domain Filtering:**
```python
INTERNAL_EMAIL_DOMAINS = ["cozycornerpatios.com", "zipcushions.com"]

def extract_customer_email(sender, recipients):
    """Extract customer email excluding internal domains"""
    all_emails = [sender] + recipients
    
    customer_emails = []
    for email_str in all_emails:
        email = extract_email_address(email_str)
        domain = email.split("@")[-1]
        
        if domain.lower() not in INTERNAL_EMAIL_DOMAINS:
            customer_emails.append(email)
    
    return customer_emails[0] if customer_emails else None
```

---

## RingCentral Integration

### Overview

Complete telephony integration with RingCentral, including OAuth authentication, call logging, recording playback, and transcription.

### Authentication Architecture

**OAuth 2.0 Flow:**
```
1. User initiates OAuth → RingCentral login page
2. User authorizes → Redirect with auth code
3. Exchange code for tokens → Access + Refresh tokens
4. Store in CRM RingCentral Settings
5. Auto-refresh when expired
```

**Token Management:**

```python
class RingCentralClient:
    def authenticate_auto(self, account_id="~"):
        """Auto-authenticate using OAuth or JWT"""
        # Try OAuth first (persistent)
        oauth_success = self.authenticate_with_oauth()
        if oauth_success:
            return True
        
        # Fall back to JWT (legacy)
        return self.authenticate_with_jwt(account_id)
    
    def authenticate_with_oauth(self):
        """Authenticate using stored OAuth tokens"""
        from crm.api.ringcentral_auth import get_valid_access_token
        
        access_token = get_valid_access_token()
        if not access_token:
            return False
        
        self.platform.auth().set_data({
            "access_token": access_token,
            "token_type": "Bearer"
        })
        return True
```

**Token Refresh:**

```python
def get_valid_access_token():
    """Get valid access token, refresh if expired"""
    settings = frappe.get_single("CRM RingCentral Settings")
    
    if not settings.access_token:
        return None
    
    # Check if token is expired
    current_time = int(time.time())
    if current_time > settings.token_expires_at:
        # Refresh token
        return refresh_access_token()
    
    return settings.access_token

def refresh_access_token():
    """Refresh OAuth access token"""
    settings = frappe.get_single("CRM RingCentral Settings")
    
    if not settings.refresh_token:
        return None
    
    # Exchange refresh token for new access token
    response = requests.post(
        "https://platform.ringcentral.com/restapi/oauth/token",
        data={
            "grant_type": "refresh_token",
            "refresh_token": settings.refresh_token,
            "client_id": settings.client_id,
            "client_secret": settings.get_password("client_secret")
        }
    )
    
    if response.status_code == 200:
        data = response.json()
        
        # Update settings
        settings.access_token = data["access_token"]
        settings.refresh_token = data.get("refresh_token", settings.refresh_token)
        settings.token_expires_at = int(time.time()) + data["expires_in"]
        settings.save()
        
        return data["access_token"]
    
    return None
```

### Benefits

- ✅ No more hourly re-authorization
- ✅ Seamless user experience
- ✅ Works for both CRM and Helpdesk
- ✅ Backward compatible with JWT

### Recording Management

**Download and Store:**

```python
def fetch_and_save_recording(call_log_name):
    """Fetch recording from RingCentral and save to ERPNext"""
    call_log = frappe.get_doc("CRM Call Log", call_log_name)
    
    if not call_log.recording_url:
        return {"success": False, "message": "No recording URL"}
    
    # Authenticate
    client = RingCentralClient()
    if not client.authenticate_auto():
        return {"success": False, "message": "Authentication failed"}
    
    # Extract recording ID
    recording_id = extract_recording_id(call_log.recording_url)
    
    # Download recording
    response = client.platform.get(
        f"/restapi/v1.0/account/~/recording/{recording_id}/content"
    )
    
    if response.status_code != 200:
        return {"success": False, "message": "Failed to download"}
    
    # Save to file system
    file_doc = frappe.get_doc({
        "doctype": "File",
        "file_name": f"recording_{call_log.name}.mp3",
        "content": response.content,
        "is_private": 1,
        "attached_to_doctype": "CRM Call Log",
        "attached_to_name": call_log.name
    })
    file_doc.insert()
    
    # Update call log
    call_log.db_set("recording_file", file_doc.file_url)
    
    return {
        "success": True,
        "file_url": file_doc.file_url
    }
```

### Transcription

**Fetch Transcript:**

```python
def get_ringcentral_transcript(self, recording_id, account_id="~"):
    """Get transcript from RingCentral API"""
    try:
        # Get recording metadata
        response = self.platform.get(
            f"/restapi/v1.0/account/{account_id}/recording/{recording_id}"
        )
        
        if response.status_code != 200:
            return None
        
        data = response.json()
        
        # Check if transcription is available
        if not data.get("transcription"):
            return None
        
        transcription_uri = data["transcription"].get("uri")
        if not transcription_uri:
            return None
        
        # Fetch transcription
        trans_response = self.platform.get(transcription_uri)
        
        if trans_response.status_code != 200:
            return None
        
        trans_data = trans_response.json()
        
        # Extract text from segments
        segments = trans_data.get("segments", [])
        transcript_text = "\n".join([
            f"[{seg.get('speakerId', 'Unknown')}]: {seg.get('text', '')}"
            for seg in segments
        ])
        
        return transcript_text
        
    except Exception as e:
        frappe.log_error(f"Error fetching transcript: {str(e)}")
        return None
```

---

## Deployment Guide

### Prerequisites

**System Requirements:**
- Python 3.10+
- Node.js 18+
- Frappe Framework v15+
- ERPNext v15+
- RingCentral account with API access
- Pabbly Connect account (for webhook routing)
- Production domain with SSL certificate
- Redis with RediSearch module installed

### Installation Steps

**1. Pull Latest Code:**
```bash
cd ~/frappe-bench
git -C apps/crm pull origin main
git -C apps/helpdesk pull origin main
git -C apps/erpnext pull origin main
```

**2. Install Dependencies:**
```bash
# Python
bench pip install ringcentral

# Node
cd apps/crm/frontend && yarn install
cd apps/helpdesk/desk && yarn install
```

**3. Run Migrations:**
```bash
bench --site mysite.local migrate
```

**4. Build Frontend:**
```bash
bench build --app crm
bench build --app helpdesk
```

**5. Clear Cache:**
```bash
bench --site mysite.local clear-cache
bench --site mysite.local clear-website-cache
```

**6. Restart Services:**
```bash
bench restart
```

---

## RingCentral Integration Setup (Production)

### Overview

This section provides complete step-by-step instructions for setting up RingCentral telephony integration on your production ERPNext instance. This setup enables:
- Automatic call logging in CRM and Helpdesk
- Call recording playback
- Call transcription viewing
- Click-to-call functionality

### Architecture

```
RingCentral Phone System
         ↓
    (Call Event)
         ↓
RingCentral Webhooks → Pabbly Connect → ERPNext Webhook Endpoint
         ↓                    ↓                    ↓
   (Filters/Routes)    (Data Transform)    (Create Call Logs)
```

---

### Part 1: RingCentral Developer Account Setup

#### Step 1.1: Create RingCentral App

**1. Log in to RingCentral Developer Portal:**
- Go to: https://developers.ringcentral.com/
- Click "Sign In" (use your RingCentral admin credentials)
- If you don't have developer access, contact your RingCentral account manager

**2. Create New App:**
- Click "Create App" or "My Apps" → "Create App"
- Choose **"REST API App"** (not Browser-based or Server-only)

**3. Configure App Settings:**

| Field | Value | Notes |
|-------|-------|-------|
| **App Name** | ERPNext CRM Integration | Or your company name |
| **App Type** | **Public** (for OAuth) or Private | Use Public for easier OAuth |
| **Platform Type** | **Server/Web** | Required for OAuth flow |
| **Carrier** | Your RingCentral carrier | Usually RingCentral US |

**4. Set OAuth Redirect URI:**
```
https://your-erp-domain.com/api/method/crm.api.ringcentral_auth.oauth_callback
```

⚠️ **CRITICAL:** Replace `your-erp-domain.com` with your actual production domain.

**Examples:**
- `https://erp.cozycornerpatios.com/api/method/crm.api.ringcentral_auth.oauth_callback`
- `https://erp.yourcompany.com/api/method/crm.api.ringcentral_auth.oauth_callback`

**5. Configure Permissions:**

Select these OAuth scopes:

**Required Scopes:**
- ✅ `ReadCallLog` - Read call history
- ✅ `ReadCallRecording` - Download recordings
- ✅ `CallControl` - Initiate calls (for click-to-call)
- ✅ `ReadAccounts` - Read account information
- ✅ `Webhook` - Receive webhook notifications (if using native webhooks)

**Optional (Recommended):**
- ✅ `ReadMessages` - Read SMS/voicemail
- ✅ `EditExtensions` - Manage extensions

**6. Save and Note Credentials:**

After creating the app, you'll see:
```
Client ID: abcd1234-ef56-7890-ghij-klmnopqrstuv
Client Secret: xyzABC123def456GHI789jkl012MNO345pqr678
```

⚠️ **IMPORTANT:** 
- Copy both immediately
- Store in secure password manager
- Client Secret is shown only once
- Never commit these to Git

#### Step 1.2: Test App in Sandbox (Optional but Recommended)

**1. Switch to Sandbox Environment:**
- In app settings, toggle "Sandbox" mode
- Use sandbox credentials for testing

**2. Test OAuth Flow:**
```bash
# Test URL format
https://platform.devtest.ringcentral.com/restapi/oauth/authorize?response_type=code&client_id=YOUR_CLIENT_ID&redirect_uri=https://your-domain.com/api/method/crm.api.ringcentral_auth.oauth_callback
```

**3. Once Sandbox Works:**
- Switch to "Production" mode in app settings
- Update credentials in ERPNext

---

### Part 2: ERPNext RingCentral Settings Configuration

#### Step 2.1: Access RingCentral Settings

**1. Navigate to Settings:**
```
ERPNext Desk → CRM Module → Settings → CRM RingCentral Settings
```

Or use Quick Search (Ctrl+K):
```
Type: "CRM RingCentral Settings"
```

#### Step 2.2: Configure Basic Settings

**Field Mapping:**

| Field | Value | Description |
|-------|-------|-------------|
| **Client ID** | Your app's Client ID | From RingCentral Developer Portal |
| **Client Secret** | Your app's Client Secret | Click "Set Password" to enter securely |
| **Server URL** | `https://platform.ringcentral.com` | Production URL (US) |
| **Environment** | Production | Don't change after initial setup |
| **Company Phone Number** | `+1234567890` | Your RingCentral main number (E.164 format) |

**Server URLs by Region:**
- **US/Canada:** `https://platform.ringcentral.com`
- **Europe:** `https://platform.eu.ringcentral.com`
- **UK:** `https://platform.ringcentral.biz`

**Phone Number Format:**
- Use E.164 format: `+[country][area][number]`
- Example: `+15551234567` (not `555-123-4567`)

#### Step 2.3: Complete OAuth Authorization

**1. Save Settings:**
- Click "Save" button (important - must save first)

**2. Authorize Button:**
- After saving, you'll see an "Authorize with RingCentral" button
- Click it to start OAuth flow

**3. OAuth Flow:**
```
Step 1: ERPNext → Redirects to RingCentral login
         ↓
Step 2: Login with RingCentral admin account
         ↓
Step 3: Authorize app to access your account
         ↓
Step 4: Redirect back to ERPNext with auth code
         ↓
Step 5: ERPNext exchanges code for tokens (automatic)
         ↓
Step 6: Tokens saved, authorization complete
```

**4. Verify Authorization:**

After successful authorization, these fields will be auto-populated:

| Field | Status | Example Value |
|-------|--------|---------------|
| **Access Token** | ✅ Filled | `U0pDMDFQMDFQQVMwM...` (long string) |
| **Refresh Token** | ✅ Filled | `U0pDMDFQMDFQQVMwM...` (long string) |
| **Token Expires At** | ✅ Filled | Unix timestamp (e.g., 1736553600) |
| **Token Expires In** | ✅ Filled | Seconds (usually 3600) |

**5. Test Connection:**

Open ERPNext Console (bench console):
```python
import frappe
from crm.integrations.ringcentral_client import RingCentralClient

# Test authentication
client = RingCentralClient()
result = client.authenticate_auto()

if result:
    print("✅ RingCentral authentication successful!")
    
    # Test API call
    response = client.platform.get('/restapi/v1.0/account/~/extension/~')
    print(f"✅ API working! Extension: {response.json()['extensionNumber']}")
else:
    print("❌ Authentication failed!")
```

#### Step 2.4: Configure Auto-Refresh (Already Built-In)

The system automatically refreshes tokens before they expire. No manual configuration needed.

**How It Works:**
1. Before any API call, system checks token expiration
2. If expired (or about to expire), refresh automatically
3. New tokens saved to database
4. API call proceeds with fresh token

**Monitor Token Refresh:**
```python
# Check token status
import frappe
settings = frappe.get_single("CRM RingCentral Settings")

import time
current_time = int(time.time())
expires_at = settings.token_expires_at

if current_time > expires_at:
    print("⚠️ Token expired!")
else:
    remaining = expires_at - current_time
    print(f"✅ Token valid for {remaining // 60} minutes")
```

---

### Part 3: Pabbly Connect Setup

#### Why Pabbly?

RingCentral webhooks have limitations:
- ❌ Limited filtering options
- ❌ Can't transform payload easily
- ❌ Single webhook URL per event
- ❌ Difficult to debug

**Pabbly Connect provides:**
- ✅ Visual workflow builder
- ✅ Data transformation/mapping
- ✅ Filtering and routing
- ✅ Error handling and retry
- ✅ Detailed logs for debugging

#### Step 3.1: Create Pabbly Connect Account

**1. Sign Up:**
- Go to: https://www.pabbly.com/connect/
- Click "Start Free Trial" or "Sign Up"
- Use company email (not personal Gmail)

**2. Choose Plan:**
- **Free Plan:** 100 tasks/month (good for testing)
- **Standard Plan:** $19/month - 12,000 tasks (recommended for production)
- **Pro Plan:** $39/month - 24,000 tasks

**Task Calculation:**
- 1 incoming call = 1 task
- If processing 500 calls/month → Standard Plan sufficient

**3. Verify Email:**
- Check inbox for verification email
- Click verification link

#### Step 3.2: Create RingCentral to ERPNext Workflow

**1. Create New Workflow:**
- Click "Create Workflow" button
- Name it: **"RingCentral to ERPNext Call Logging"**

**2. Set Up Trigger (RingCentral Webhook):**

**Step 2a: Add Trigger Step**
- Click "Trigger" → Search "RingCentral"
- Select **"RingCentral"** app
- Choose trigger event: **"New Call Completed"** or **"Call Log Event"**

**Step 2b: Connect RingCentral Account**
- Click "Connect"
- Choose **"OAuth 2.0"** method
- You'll be redirected to RingCentral
- Login with admin credentials
- Authorize Pabbly to access call data
- Redirect back to Pabbly

**Step 2c: Configure Trigger Settings**
```json
{
  "Event": "Call Completed",
  "Direction": "All" (or filter: "Inbound" / "Outbound"),
  "Extension": "All" (or specific extension)
}
```

**Step 2d: Test Trigger**
- Click "Send Test Request"
- Make a test call to your RingCentral number
- Wait 10-30 seconds
- Click "Refresh" in Pabbly
- You should see call data appear

**Example Trigger Data:**
```json
{
  "id": "12345678",
  "sessionId": "87654321",
  "startTime": "2025-01-06T10:30:00Z",
  "duration": 120,
  "direction": "Inbound",
  "from": {
    "phoneNumber": "+15551234567",
    "name": "John Doe"
  },
  "to": {
    "phoneNumber": "+15559876543",
    "name": "Support Line"
  },
  "recording": {
    "id": "rec-12345",
    "contentUri": "https://platform.ringcentral.com/restapi/v1.0/account/~/recording/rec-12345/content"
  },
  "legs": [...]
}
```

#### Step 3.3: Add Data Transformation (Optional but Recommended)

**1. Add Router/Filter Step:**
- Click "+" after trigger
- Choose "Router" → "Filter"
- Configure filter:

**Filter Examples:**

**Filter 1: Only Answered Calls**
```
Field: result
Condition: Equal To
Value: Call connected
```

**Filter 2: Minimum Duration (skip missed calls)**
```
Field: duration
Condition: Greater Than
Value: 5
```

**Filter 3: Skip Internal Calls**
```
Field: from.phoneNumber
Condition: Does Not Contain
Value: [your internal extension pattern]
```

**2. Add Data Formatter (Optional):**
- Click "+" → Choose "Formatter"
- Select "Date/Time Formatter"
- Format: Convert RingCentral timestamp to ISO format

#### Step 3.4: Configure ERPNext Webhook Action

**1. Add Action Step:**
- Click "+" after trigger/filter
- Search "Webhooks by Pabbly"
- Select **"POST Request"**

**2. Configure Webhook URL:**
```
https://your-erp-domain.com/api/method/helpdesk.api.ringcentral_webhook.pabbly_webhook
```

⚠️ **Replace with your actual domain:**
- Example: `https://erp.cozycornerpatios.com/api/method/helpdesk.api.ringcentral_webhook.pabbly_webhook`

**3. Set Request Method:**
- Method: **POST**
- Content-Type: **application/json**

**4. Configure Authentication:**

**Option A: API Key/Secret (Recommended)**

Add headers:
```json
{
  "Authorization": "token YOUR_API_KEY:YOUR_API_SECRET"
}
```

To generate API key in ERPNext:
```
User Menu → API Access → Generate Keys
```

**Option B: No Authentication (Less Secure)**
- Only if ERPNext endpoint is whitelisted (not recommended for production)

**5. Map Request Body:**

Click "Add New Field" for each field you want to send:

**Basic Mapping:**

| Pabbly Field Name | Map From RingCentral | Example |
|-------------------|----------------------|---------|
| `call_id` | `id` | "12345678" |
| `session_id` | `sessionId` | "87654321" |
| `direction` | `direction` | "Inbound" |
| `from_number` | `from.phoneNumber` | "+15551234567" |
| `from_name` | `from.name` | "John Doe" |
| `to_number` | `to.phoneNumber` | "+15559876543" |
| `to_name` | `to.name` | "Support Line" |
| `start_time` | `startTime` | "2025-01-06T10:30:00Z" |
| `duration` | `duration` | 120 |
| `result` | `result` | "Call connected" |
| `recording_id` | `recording.id` | "rec-12345" |
| `recording_url` | `recording.contentUri` | "https://..." |

**Complete JSON Body Example:**
```json
{
  "call_id": "{{1.id}}",
  "session_id": "{{1.sessionId}}",
  "direction": "{{1.direction}}",
  "from_number": "{{1.from.phoneNumber}}",
  "from_name": "{{1.from.name}}",
  "to_number": "{{1.to.phoneNumber}}",
  "to_name": "{{1.to.name}}",
  "start_time": "{{1.startTime}}",
  "duration": "{{1.duration}}",
  "result": "{{1.result}}",
  "recording": {
    "id": "{{1.recording.id}}",
    "contentUri": "{{1.recording.contentUri}}"
  },
  "legs": "{{1.legs}}"
}
```

**Note:** `{{1.field}}` syntax pulls data from step 1 (trigger)

**6. Test Webhook:**
- Click "Send Test Request"
- Check response from ERPNext:

**Success Response:**
```json
{
  "success": true,
  "message": "Call log created",
  "call_log_name": "HD-CALL-00001"
}
```

**Error Response:**
```json
{
  "success": false,
  "message": "Error: Field 'from_number' is required"
}
```

**7. Save and Activate Workflow:**
- Click "Save" button
- Toggle "Active" switch to ON
- Workflow is now live!

#### Step 3.5: Test End-to-End Flow

**1. Make Test Call:**
- Call your RingCentral number from external phone
- Talk for at least 10 seconds
- Hang up

**2. Wait for Processing:**
- RingCentral webhook fires (1-5 seconds delay)
- Pabbly receives webhook (should see in logs)
- Pabbly sends to ERPNext (1-2 seconds)
- ERPNext creates call log

**3. Verify in ERPNext:**

**Check Helpdesk Call Logs:**
```
Helpdesk → Call Logs → You should see new entry
```

**Verify Fields:**
- ✅ From Number: Matches caller
- ✅ To Number: Matches RingCentral line
- ✅ Duration: Correct
- ✅ Direction: Inbound
- ✅ Recording URL: Present (if recorded)
- ✅ Ticket: Auto-created or linked (if customer exists)

**4. Check Pabbly Logs:**
```
Pabbly Connect → Workflow History → Recent Executions
```

Look for:
- ✅ Green checkmark = Success
- ❌ Red X = Failed (click to see error)

#### Step 3.6: Advanced Pabbly Configuration

**A. Handle Call Recording Availability:**

RingCentral recordings may not be available immediately. Create a second workflow:

**Workflow 2: "RingCentral Recording Available"**

**Trigger:** RingCentral → "Recording Ready" event

**Action:** Send to ERPNext endpoint with recording data
```
POST https://your-domain.com/api/method/helpdesk.api.ringcentral_webhook.recording_webhook
```

**Body:**
```json
{
  "call_log_id": "{{call_id}}",
  "recording_id": "{{recording.id}}",
  "recording_url": "{{recording.contentUri}}"
}
```

**B. Error Handling in Pabbly:**

Add error handling steps:

**1. Add Router After Webhook:**
- If response status = 200 → Success path
- If response status != 200 → Error path

**2. Error Path Actions:**
- Send email notification to admin
- Log to Google Sheets for tracking
- Retry after 5 minutes (use delay step)

**C. Rate Limiting:**

If you have high call volume:

**1. Add Delay Step:**
- Click "+" → "Delay"
- Set delay: 1 second between calls
- Prevents overwhelming ERPNext

**2. Configure Batch Processing:**
- Accumulate calls in array
- Send batch every 10 calls or 5 minutes

---

### Part 4: ERPNext Webhook Endpoint Configuration

#### Step 4.1: Verify Webhook Endpoint is Whitelisted

**Check File:**
```python
# File: apps/helpdesk/helpdesk/api/ringcentral_webhook.py

@frappe.whitelist(allow_guest=True)  # ← Must have this
def pabbly_webhook():
    """Receive webhooks from Pabbly Connect"""
    # ... code ...
```

If `allow_guest=True` is missing, add it and restart:
```bash
bench restart
```

#### Step 4.2: Configure Webhook Security (Recommended)

**Option 1: API Key Authentication**

Update webhook to require authentication:

```python
@frappe.whitelist()  # Remove allow_guest
def pabbly_webhook():
    """Receive webhooks from Pabbly Connect (authenticated)"""
    # Authentication is handled by @frappe.whitelist()
    # ... rest of code ...
```

Then in Pabbly, add header:
```
Authorization: token YOUR_API_KEY:YOUR_API_SECRET
```

**Option 2: Secret Token Validation**

Add custom secret validation:

```python
@frappe.whitelist(allow_guest=True)
def pabbly_webhook():
    """Receive webhooks from Pabbly Connect"""
    # Validate secret token
    secret = frappe.get_request_header("X-Webhook-Secret")
    expected_secret = frappe.db.get_single_value("Helpdesk Settings", "webhook_secret")
    
    if secret != expected_secret:
        frappe.throw("Invalid webhook secret", frappe.PermissionError)
    
    # ... rest of code ...
```

Add custom field to Helpdesk Settings:
```
Field: webhook_secret
Type: Password
```

Set value to random string (e.g., `sk_live_abc123def456`)

In Pabbly, add header:
```
X-Webhook-Secret: sk_live_abc123def456
```

#### Step 4.3: Test Webhook Directly (Without Pabbly)

Use curl to test:

```bash
curl -X POST https://your-domain.com/api/method/helpdesk.api.ringcentral_webhook.pabbly_webhook \
  -H "Content-Type: application/json" \
  -H "Authorization: token YOUR_API_KEY:YOUR_API_SECRET" \
  -d '{
    "call_id": "test-123",
    "direction": "Inbound",
    "from_number": "+15551234567",
    "from_name": "Test Caller",
    "to_number": "+15559876543",
    "start_time": "2025-01-06T10:30:00Z",
    "duration": 120,
    "result": "Call connected",
    "recording": {
      "id": "rec-test-123",
      "contentUri": "https://example.com/recording.mp3"
    }
  }'
```

Expected response:
```json
{
  "success": true,
  "message": "Call log created",
  "call_log_name": "HD-CALL-00001"
}
```

---

### Part 5: Production Deployment Checklist

#### Pre-Deployment

- [ ] RingCentral app created in Production mode (not Sandbox)
- [ ] Client ID and Client Secret saved securely
- [ ] OAuth redirect URI matches production domain exactly
- [ ] All required OAuth scopes enabled
- [ ] ERPNext RingCentral Settings configured
- [ ] OAuth authorization completed successfully
- [ ] Token refresh tested and working
- [ ] Pabbly Connect account created (paid plan for production)
- [ ] Pabbly workflow created and tested
- [ ] Pabbly to ERPNext webhook tested successfully
- [ ] End-to-end call flow tested (make real call)
- [ ] Webhook endpoint secured (API key or secret)
- [ ] SSL certificate valid on ERPNext domain
- [ ] Firewall allows incoming webhooks from Pabbly IPs

#### Deployment Steps

**1. Pull Latest Code to Production:**
```bash
cd ~/frappe-bench
git -C apps/crm pull origin main
git -C apps/helpdesk pull origin main
```

**2. Install RingCentral SDK:**
```bash
bench pip install ringcentral
```

**3. Run Migrations:**
```bash
bench --site your-site.com migrate
```

**4. Build Frontend:**
```bash
bench build --app crm
bench build --app helpdesk
```

**5. Clear Cache:**
```bash
bench --site your-site.com clear-cache
```

**6. Restart Services:**
```bash
bench restart
```

**7. Configure Settings:**
- Go to CRM RingCentral Settings
- Enter production Client ID and Secret
- Complete OAuth authorization
- Verify tokens are saved

**8. Activate Pabbly Workflow:**
- Switch workflow to "Active"
- Monitor first few calls closely

**9. Test with Real Call:**
- Make test call to RingCentral number
- Verify call log created in ERPNext
- Check recording and transcript (if available)
- Verify ticket creation (for new customers)

#### Post-Deployment Monitoring

**Day 1: Hourly Checks**
```python
# Check recent call logs
import frappe
logs = frappe.get_all(
    "Helpdesk Call Log",
    filters={"creation": [">=", frappe.utils.add_days(frappe.utils.nowdate(), -1)]},
    fields=["name", "from_number", "to_number", "duration", "creation"],
    order_by="creation desc",
    limit=20
)

for log in logs:
    print(f"{log.creation}: {log.name} - {log.from_number} → {log.to_number} ({log.duration}s)")
```

**Day 1-7: Daily Checks**
- Review Pabbly execution logs
- Check ERPNext error logs
- Verify all calls are being logged
- Monitor token refresh (should be automatic)
- Check recording downloads are working

**Week 2+: Weekly Checks**
- Review call volume statistics
- Check for any failed webhook deliveries
- Verify recording storage isn't filling up disk
- Monitor API usage (RingCentral has limits)

#### Troubleshooting Commands

**Check RingCentral Token Status:**
```python
import frappe
settings = frappe.get_single_value("CRM RingCentral Settings")
print(f"Access Token: {settings.access_token[:20]}...")
print(f"Expires At: {settings.token_expires_at}")
```

**Force Token Refresh:**
```python
from crm.api.ringcentral_auth import refresh_access_token
new_token = refresh_access_token()
print(f"New Token: {new_token[:20]}...")
```

**Test API Connection:**
```python
from crm.integrations.ringcentral_client import RingCentralClient
client = RingCentralClient()
if client.authenticate_auto():
    print("✅ Connected!")
else:
    print("❌ Failed!")
```

**Check Recent Webhooks:**
```python
import frappe
logs = frappe.get_all(
    "Error Log",
    filters={"method": ["like", "%ringcentral_webhook%"]},
    fields=["name", "error", "creation"],
    order_by="creation desc",
    limit=10
)
```

---

### Part 6: Pabbly IP Whitelisting (If Firewall Enabled)

If your ERPNext server has firewall rules, whitelist Pabbly IPs:

**Pabbly Connect IP Addresses:**
```
52.90.172.94
54.163.39.217
54.165.110.233
54.173.237.155
```

**Whitelist in UFW (Ubuntu):**
```bash
sudo ufw allow from 52.90.172.94 to any port 443
sudo ufw allow from 54.163.39.217 to any port 443
sudo ufw allow from 54.165.110.233 to any port 443
sudo ufw allow from 54.173.237.155 to any port 443
sudo ufw reload
```

**Whitelist in AWS Security Group:**
- Add inbound rule: HTTPS (443) from these IPs

---

### Part 7: RingCentral API Rate Limits

**Rate Limits (as of 2025):**
- **Light:** 10 requests/minute/user
- **Medium:** 40 requests/minute/user  
- **Heavy:** 50 requests/minute/user (webhooks, recordings)

**Best Practices:**
- Cache recordings in ERPNext (don't fetch repeatedly)
- Cache transcripts in database
- Use webhooks instead of polling
- Implement exponential backoff on errors

**Monitor Rate Limit:**
```python
# Check response headers
response = client.platform.get('/restapi/v1.0/account/~/extension/~')
print(f"Rate Limit: {response.headers.get('X-Rate-Limit-Limit')}")
print(f"Remaining: {response.headers.get('X-Rate-Limit-Remaining')}")
print(f"Reset: {response.headers.get('X-Rate-Limit-Window')}")
```

---

### Part 8: Security Best Practices

**1. Store Credentials Securely:**
- ✅ Use Password fields (encrypted in database)
- ✅ Never log Client Secret or tokens
- ✅ Restrict access to RingCentral Settings (only admins)

**2. Webhook Security:**
- ✅ Use HTTPS only (never HTTP)
- ✅ Validate webhook signatures or use secret token
- ✅ Rate limit webhook endpoint
- ✅ Log all webhook attempts

**3. API Key Management:**
- ✅ Create dedicated API user for webhooks
- ✅ Limit permissions (only create call logs)
- ✅ Rotate API keys quarterly
- ✅ Monitor API usage

**4. Recording Storage:**
- ✅ Store recordings in private files (is_private=1)
- ✅ Encrypt at rest (if possible)
- ✅ Implement retention policy (delete after X days)
- ✅ Restrict access to authorized users only

**5. Compliance:**
- ✅ Add call recording disclosure (legal requirement in some states)
- ✅ Honor do-not-call lists
- ✅ GDPR compliance: Allow customers to request deletion
- ✅ Log retention compliance (keep for X years)

---

### Part 9: Advanced Configuration

#### Multi-Site Setup

If you have multiple ERPNext sites:

**1. Create Site-Specific Webhook URLs:**
```
Site 1: https://erp1.company.com/api/method/helpdesk.api.ringcentral_webhook.pabbly_webhook
Site 2: https://erp2.company.com/api/method/helpdesk.api.ringcentral_webhook.pabbly_webhook
```

**2. Use Pabbly Router:**
- Add Router step after RingCentral trigger
- Route based on called number or extension
- Send to appropriate site

#### Multi-Company Setup

**1. Configure Company Phone Numbers:**
```python
# In CRM RingCentral Settings
Company 1: +15551111111
Company 2: +15552222222
```

**2. Route by Called Number:**
- Pabbly checks `to.phoneNumber`
- Routes to correct company's webhook

#### Call Recording Storage Options

**Option 1: Store in ERPNext (Default)**
- Recordings stored in `frappe-bench/sites/your-site/private/files`
- Accessible via ERPNext File Manager
- Backed up with site backups

**Option 2: Store in S3**
```python
# Configure S3 in site_config.json
{
  "s3_backup_bucket": "your-bucket",
  "aws_access_key_id": "YOUR_KEY",
  "aws_secret_access_key": "YOUR_SECRET"
}
```

**Option 3: Keep in RingCentral Only**
- Don't download recordings
- Link to RingCentral URL only
- Saves storage space
- Recordings auto-delete after 90 days (RingCentral policy)

---

### Configuration Summary

**✅ Complete Setup Checklist:**

**RingCentral:**
- [ ] Developer account created
- [ ] App created with OAuth
- [ ] Redirect URI configured
- [ ] OAuth scopes enabled
- [ ] Client ID and Secret obtained

**ERPNext:**
- [ ] Code pulled and deployed
- [ ] RingCentral SDK installed
- [ ] Migrations run
- [ ] Frontend built
- [ ] RingCentral Settings configured
- [ ] OAuth authorization completed
- [ ] Token refresh working

**Pabbly:**
- [ ] Account created (paid plan)
- [ ] RingCentral connected
- [ ] Workflow created
- [ ] Data mapping configured
- [ ] ERPNext webhook configured
- [ ] Workflow activated
- [ ] End-to-end tested

**Security:**
- [ ] SSL enabled on ERPNext
- [ ] Webhook authentication enabled
- [ ] Firewall configured (if needed)
- [ ] Credentials stored securely
- [ ] Access restricted to admins

**Testing:**
- [ ] Test call made and logged
- [ ] Recording playback works
- [ ] Transcript fetching works
- [ ] Ticket auto-creation works
- [ ] Click-to-call works

**Production Ready! 🚀**

---

## Configuration Examples by Company Size

### Small Business (< 50 calls/day)

**RingCentral Plan:** Essentials  
**Pabbly Plan:** Free (100 tasks/month) or Standard ($19/month)  
**ERPNext Resources:** 2GB RAM, 1 CPU sufficient  

**Recommended Setup:**
- Single workflow for all calls
- Store recordings in ERPNext
- No advanced filtering needed

### Medium Business (50-500 calls/day)

**RingCentral Plan:** Standard or Premium  
**Pabbly Plan:** Standard ($19/month) or Pro ($39/month)  
**ERPNext Resources:** 4GB RAM, 2 CPU recommended  

**Recommended Setup:**
- Separate workflows for inbound/outbound
- Filter missed calls
- Store recordings in S3
- Implement call routing by department

### Enterprise (> 500 calls/day)

**RingCentral Plan:** Ultimate  
**Pabbly Plan:** Ultimate ($249/month - 300k tasks)  
**ERPNext Resources:** 8GB+ RAM, 4+ CPU  

**Recommended Setup:**
- Multiple workflows by department
- Advanced filtering and routing
- S3 storage with CDN
- Dedicated API user
- Load balancing for webhooks
- Real-time dashboards

---

**Setup complete! Your RingCentral telephony integration is now production-ready.**

---

## API Reference

### CRM APIs

**Lead Auto-Merge:**
```python
# Automatic (triggered on save)
lead = frappe.get_doc("CRM Lead", lead_name)
lead.mobile_no = "+15551234567"
lead.save()  # Triggers auto-merge
```

**Email Transfer:**
```python
# Transfer to Helpdesk
frappe.call(
    method='crm.api.email_transfer.transfer_to_helpdesk',
    args={
        'lead_name': 'LEAD-2025-00001',
        'communication_ids': ['COMM-001', 'COMM-002'],
        'delete_source': True
    }
)

# Get communications for transfer
frappe.call(
    method='crm.api.email_transfer.get_communications_for_transfer',
    args={
        'doctype': 'CRM Lead',
        'name': 'LEAD-2025-00001'
    }
)
```

**Order History:**
```python
# Fetch lead order history
frappe.call(
    method='crm.api.order_history.fetch_lead_order_history',
    args={
        'lead_name': 'LEAD-2025-00001',
        'customer_name': 'John Doe'
    }
)
```

**Call Transcription:**
```python
# Get or create transcript
frappe.call(
    method='crm.api.call_transcription.get_or_create_transcript',
    args={'call_log_name': 'CALL-2025-00001'}
)
```

### Helpdesk APIs

**Click-to-Call:**
```python
# Create manual call log
frappe.call(
    method='helpdesk.api.ringcentral_calling.create_manual_call_log',
    args={
        'phone_number': '+15551234567',
        'contact_name': 'John Doe',
        'ticket_name': 'HD-TICKET-00001'
    }
)
```

**Recording Playback:**
```python
# Fetch and save recording
frappe.call(
    method='helpdesk.api.recording.fetch_and_save_helpdesk_recording',
    args={'call_log_name': 'HD-CALL-00001'}
)
```

**Phone Merging:**
```python
# Manual merge
frappe.call(
    method='helpdesk.api.ticket_phone_utils.merge_call_logs_by_phone',
    args={'ticket_name': 'HD-TICKET-00001'}
)

# Find duplicates
frappe.call(
    method='helpdesk.api.ticket_phone_utils.find_duplicate_tickets_by_phone',
    args={'phone_number': '+15551234567'}
)

# Bulk merge
frappe.call(
    method='helpdesk.api.ticket_phone_utils.bulk_merge_duplicate_tickets'
)
```

---

## Troubleshooting

### Common Issues

**Issue: RingCentral Authentication Fails**

**Solutions:**
```bash
# Check OAuth tokens
bench --site mysite.local console
>>> import frappe
>>> settings = frappe.get_single("CRM RingCentral Settings")
>>> print(f"Access Token: {settings.access_token[:20]}...")
>>> print(f"Expires At: {settings.token_expires_at}")

# Force token refresh
>>> from crm.api.ringcentral_auth import refresh_access_token
>>> new_token = refresh_access_token()
>>> print(f"New Token: {new_token[:20]}...")
```

**Issue: Call Logs Not Created**

**Solutions:**
```python
# Check webhook logs
import frappe
logs = frappe.get_all(
    "Error Log",
    filters={"method": ["like", "%ringcentral_webhook%"]},
    fields=["name", "error", "creation"],
    order_by="creation desc",
    limit=10
)

for log in logs:
    print(f"{log.creation}: {log.error[:100]}")
```

**Issue: Phone Merging Not Working**

**Solutions:**
```python
# Check phone normalization
from crm.integrations.ringcentral_utils import normalize_phone_number

phone = "+1 (555) 123-4567"
normalized = normalize_phone_number(phone)
print(f"{phone} → {normalized}")

# Find duplicates manually
import frappe
leads = frappe.db.sql("""
    SELECT name, lead_name, mobile_no
    FROM `tabCRM Lead`
    WHERE mobile_no = %s
""", normalized, as_dict=True)

print(f"Found {len(leads)} leads with phone {normalized}")
```

---

## File Structure

```
frappe-bench/
├── apps/
│   ├── crm/
│   │   ├── crm/
│   │   │   ├── api/
│   │   │   │   ├── call_transcription.py
│   │   │   │   ├── email_transfer.py
│   │   │   │   ├── order_history.py
│   │   │   │   └── ringcentral_auth.py
│   │   │   ├── fcrm/
│   │   │   │   └── doctype/
│   │   │   │       ├── crm_lead/
│   │   │   │       │   ├── crm_lead.py
│   │   │   │       │   └── crm_lead.json
│   │   │   │       ├── crm_deal/
│   │   │   │       │   ├── crm_deal.py
│   │   │   │       │   └── crm_deal.json
│   │   │   │       └── crm_deal_sales_orders/
│   │   │   │           ├── crm_deal_sales_orders.py
│   │   │   │           └── crm_deal_sales_orders.json
│   │   │   └── integrations/
│   │   │       ├── ringcentral_client.py
│   │   │       └── ringcentral_utils.py
│   │   └── frontend/
│   │       └── src/
│   │           ├── components/
│   │           │   ├── EmailTransferDialog.vue
│   │           │   └── CallArea.vue
│   │           └── utils/
│   │               └── emailTransfer.js
│   │
│   ├── helpdesk/
│   │   ├── helpdesk/
│   │   │   ├── api/
│   │   │   │   ├── ringcentral_calling.py
│   │   │   │   ├── recording.py
│   │   │   │   ├── transcript.py
│   │   │   │   ├── email_transfer.py
│   │   │   │   ├── order_history.py
│   │   │   │   ├── ticket_phone_utils.py
│   │   │   │   └── ringcentral_webhook.py
│   │   │   ├── helpdesk/
│   │   │   │   └── doctype/
│   │   │   │       └── hd_ticket/
│   │   │   │           ├── hd_ticket.py
│   │   │   │           └── hd_ticket.json
│   │   │   └── integrations/
│   │   │       └── ringcentral_utils.py
│   │   └── desk/
│   │       └── src/
│   │           └── components/
│   │               ├── RingCentralCallUI.vue
│   │               ├── CallRecordingPlayer.vue
│   │               ├── CallTranscriptModal.vue
│   │               └── ticket/
│   │                   ├── TicketAgentContact.vue
│   │                   ├── TicketCallLogs.vue
│   │                   └── TicketOrderDetails.vue
│   │
│   └── erpnext/
│       └── erpnext/
│           └── crm/
│               └── doctype/
│                   └── ringcentral_crm_payload/
│                       └── ringcentral_crm_payload.py
```

---

## Email Routing System (Server Script)

### Overview

The Email Routing System is a sophisticated ERPNext Server Script that automatically routes incoming emails to either HD Tickets (Support) or CRM Leads (Sales) based on intelligent business logic. This is a critical component that ensures proper customer communication management.

**Location:** ERPNext → Automation → Server Script → "Communication Before Save"

**Trigger:** `before_save` event on `Communication` DocType

### Business Logic Flow

```
                        Incoming Email
                              ↓
                    ┌─────────────────────┐
                    │ Is External Sender? │
                    └─────────────────────┘
                         ↓ Yes
                    ┌─────────────────────┐
                    │ Is Blocked/System?  │
                    └─────────────────────┘
                         ↓ No
                    ┌─────────────────────┐
                    │ Sent to support@?   │──Yes──→ Create/Link HD Ticket
                    └─────────────────────┘
                         ↓ No
                    ┌─────────────────────┐
                    │ Reply to Ticket?    │──Yes──→ Link to HD Ticket
                    └─────────────────────┘
                         ↓ No
                    ┌─────────────────────┐
                    │ Sent to sales@      │
                    │ or info@?           │──Yes──→ Create/Link CRM Lead
                    └─────────────────────┘
                         ↓ No
                    ┌─────────────────────┐
                    │ Has Ticket History? │──Yes──→ Create HD Ticket
                    └─────────────────────┘
                         ↓ No
                    ┌─────────────────────┐
                    │  New Customer       │──────→ Create CRM Lead
                    └─────────────────────┘
```

### Key Features

#### 1. Email Classification

**Email Address → Routing Destination:**

| Email Address | Routing Destination | Priority | Behavior |
|---------------|-------------------|----------|----------|
| `support@cozycornerpatios.com` | HD Ticket | Highest | Always creates ticket |
| `support@zipcushions.com` | HD Ticket | Highest | Always creates ticket |
| `sales@cozycornerpatios.com` | CRM Lead | High | Creates lead for new customers |
| `sales@zipcushions.com` | CRM Lead | High | Creates lead for new customers |
| `info@cozycornerpatios.com` | CRM Lead | High | Creates lead for new customers |
| `info@zipcushions.com` | CRM Lead | High | Creates lead for new customers |
| Other addresses | Customer History | Medium | Checks history to decide |

**Support Emails (Priority 1):**
- `support@cozycornerpatios.com`
- `support@zipcushions.com`

**Sales Emails (Priority 2):**
- `sales@cozycornerpatios.com`
- `sales@zipcushions.com`
- `info@cozycornerpatios.com`
- `info@zipcushions.com`

#### 2. Sender Validation

**Blocked Emails:**
```python
BLOCKED_EMAILS = [
    "admin@example.com",
    "administrator@example.com",
    "admin@cozycornerpatios.com",
    "administrator@cozycornerpatios.com",
    "admin@zipcushions.com",
    "administrator@zipcushions.com"
]
```

**System Keywords:**
```python
system_keywords = [
    'mailer-daemon', 'noreply', 'no-reply', 
    'postmaster', 'bounce', 'donotreply', 
    'do-not-reply', 'automated'
]
```

#### 3. Smart Name Generation

**Name Extraction Logic:**
```python
# Priority 1: Use sender_full_name from email
if doc.sender_full_name:
    sender_name = doc.sender_full_name

# Priority 2: Extract from email address
else:
    sender_name = sender.split("@")[0].replace(".", " ").replace("_", " ").title()

# Filter out system names
if sender_name.lower() in ["administrator", "admin", "noreply", "support", "sales", "info"]:
    sender_name = "Customer"  # Generic fallback
```

#### 4. Reply Thread Detection

**Three-Level Detection:**

**Level 1: In-Reply-To Header**
```python
if doc.in_reply_to:
    existing_ticket = frappe.db.get_value(
        "Communication",
        {
            "message_id": doc.in_reply_to,
            "reference_doctype": "HD Ticket"
        },
        "reference_name"
    )
```

**Level 2: References Header**
```python
if doc.references:
    references = doc.references.split()
    for ref_id in references:
        existing_ticket = frappe.db.get_value(
            "Communication",
            {"message_id": ref_id, "reference_doctype": "HD Ticket"},
            "reference_name"
        )
```

**Level 3: Recent Open Tickets**
```python
recent_ticket = frappe.db.get_value(
    "HD Ticket",
    {
        "raised_by": sender,
        "status": ["in", ["Open", "Replied", "Waiting for Customer", "On Hold"]]
    },
    "name",
    order_by="modified desc"
)
```

### Routing Rules

#### Rule 1: Support Email (Highest Priority)

```python
sent_to_support = (
    'support@cozycornerpatios.com' in recipient or
    'support@zipcushions.com' in recipient
)

if sent_to_support:
    # ALWAYS create or link to HD Ticket
```

**Actions:**
1. Check for existing open ticket by `raised_by` email
2. If found: Link to existing ticket
3. If not found: Create new HD Ticket

**Special Handling:**
- No description field set (prevents duplicate communication)
- Contact is ensured to exist before ticket creation
- Sets `via_customer_portal = 0` to prevent auto-response

#### Rule 2: Reply to Existing Ticket

```python
if existing_ticket:
    doc.reference_doctype = "HD Ticket"
    doc.reference_name = existing_ticket
```

**Detection Methods:**
1. In-Reply-To header matches Communication.message_id
2. References header contains previous message IDs
3. Customer has open ticket within last 7 days

#### Rule 3: Sales/Info Email

```python
sent_to_sales = (
    'sales@cozycornerpatios.com' in recipient or 
    'sales@zipcushions.com' in recipient
)
sent_to_info = (
    'info@cozycornerpatios.com' in recipient or 
    'info@zipcushions.com' in recipient
)

if sent_to_sales or sent_to_info:
    # Route to CRM Lead
```

**Actions:**
1. Check for existing CRM Lead by email
2. If found: Link to existing lead
3. If not found: Create new CRM Lead with status "New"

#### Rule 4: Fallback - Customer History

```python
has_any_ticket = frappe.db.exists("HD Ticket", {"raised_by": sender})

if has_any_ticket:
    # Customer has support history → Create HD Ticket
else:
    # New customer → Create CRM Lead
```

### Critical Features

#### 1. Duplicate Communication Prevention

**Problem:** HD Ticket's `after_insert` hook creates a communication from description field, causing duplicates.

**Solution:**
```python
# CRITICAL FIX: Create ticket WITHOUT description
ticket = frappe.get_doc({
    "doctype": "HD Ticket",
    "subject": subject,
    "raised_by": sender,
    "status": "Open", 
    "priority": "Medium",
    "via_customer_portal": 0  # Prevents auto-creation
})
```

The incoming email Communication becomes the ONLY communication.

#### 2. Contact Pre-Creation

**Problem:** HD Ticket expects Contact to exist before ticket creation.

**Solution:**
```python
def ensure_contact_exists(email_addr, full_name):
    """Ensure contact exists before creating HD Ticket"""
    contact = frappe.db.get_value("Contact", {"email_id": email_addr}, "name")
    
    if not contact:
        contact_doc = frappe.get_doc({
            "doctype": "Contact",
            "first_name": full_name,
            "email_ids": [{"email_id": email_addr, "is_primary": 1}]
        })
        contact_doc.insert(ignore_permissions=True)
        return contact_doc.name
    return contact
```

Called before creating any HD Ticket.

#### 3. Subject Truncation

**Problem:** HD Ticket subject field has 140 character limit.

**Solution:**
```python
if len(subject) > 140:
    subject = subject[:137] + "..."
```

#### 4. Skip Conditions

**Must Skip If:**
```python
# Skip outgoing emails
if doc.sent_or_received != "Received":
    return

# Skip internal emails
if doc.sender.endswith("@cozycornerpatios.com"):
    return

# Skip already-routed emails
if doc.reference_doctype and doc.reference_name:
    return

# Skip system-generated (no email_account)
if not doc.email_account:
    return
```

#### 5. Comprehensive Error Logging

**Every action is logged:**
```python
frappe.log_error(
    f"[START] {sender[:30]} | {subject[:30]}",
    "Route: Start"
)

frappe.log_error(
    f"[SUPPORT] {sender}",
    "Route: Support"
)

frappe.log_error(
    f"[CREATED] Ticket: {ticket.name} | {sender}",
    "Route: Support Created"
)
```

**Log Categories:**
- `Route: Start` - Processing started
- `Route: Blocked` - Sender blocked
- `Route: System Email` - System email detected
- `Route: Name Gen` - Name generated
- `Route: Support` - Routed to support
- `Route: Support Linked` - Linked to existing ticket
- `Route: Support Created` - New ticket created
- `Route: CRM Linked` - Linked to existing lead
- `Route: CRM Created` - New lead created
- `Route: Critical Error` - Fatal error (but doesn't block)

### Error Handling

**Principle:** Never block email processing, even on errors.

```python
try:
    # Routing logic...
except Exception as e:
    # CRITICAL: Never block email processing
    error_msg = f"Communication: {doc_name}\nSender: {doc.sender}\nError: {str(e)}"
    frappe.log_error(error_msg, "Route: Critical Error")
    pass  # Allow email to save without routing
```

**Multiple Try-Catch Blocks:**
1. Main routing logic
2. Support ticket creation
3. Reply detection
4. CRM lead creation
5. Fallback routing
6. Contact creation

Each block fails gracefully and logs errors.

### Installation

**1. Create Server Script:**

Go to: **ERPNext → Automation → Server Script**

**Fields:**
- **Name:** Communication Before Save
- **DocType:** Communication
- **Event:** Before Save
- **Script Type:** DocType Event
- **Enabled:** ✓

**2. Paste Script Code**

Copy the entire script into the script field.

**3. Save and Test**

Send test emails to verify routing.

### Testing Checklist

**Support Email Tests:**
- [ ] New email to support@ creates HD Ticket
- [ ] Reply to ticket email links to existing ticket
- [ ] Multiple emails from same customer create one ticket
- [ ] Support emails never create CRM Leads

**Sales Email Tests:**
- [ ] New email to sales@ creates CRM Lead
- [ ] New email to info@ creates CRM Lead
- [ ] Reply to lead email links to existing lead
- [ ] Sales emails never create HD Tickets (unless support history)

**Reply Detection Tests:**
- [ ] Reply with In-Reply-To header links correctly
- [ ] Reply with References header links correctly
- [ ] Reply to 7-day-old ticket links correctly
- [ ] Reply to closed ticket creates new ticket

**Customer History Tests:**
- [ ] Support customer emailing info@ gets HD Ticket
- [ ] Sales customer emailing support@ gets HD Ticket
- [ ] New customer emailing unknown address gets CRM Lead

**Edge Cases:**
- [ ] Email from admin@example.com is blocked
- [ ] Email from noreply@ is blocked
- [ ] Email with no subject works
- [ ] Email with 200-char subject is truncated
- [ ] Internal email is skipped
- [ ] System-generated email is skipped

### Monitoring

**Check Error Logs:**
```python
# In bench console
import frappe
logs = frappe.get_all(
    "Error Log",
    filters={"error": ["like", "%Route:%"]},
    fields=["name", "creation", "error"],
    order_by="creation desc",
    limit=50
)

for log in logs:
    print(f"{log.creation}: {log.error[:100]}")
```

**Check Routing Statistics:**
```python
# Get routing stats for last 24 hours
from frappe.utils import add_days, nowdate

yesterday = add_days(nowdate(), -1)

# Count HD Tickets created
tickets = frappe.db.count("HD Ticket", filters={
    "creation": [">=", yesterday]
})

# Count CRM Leads created  
leads = frappe.db.count("CRM Lead", filters={
    "creation": [">=", yesterday]
})

print(f"Last 24h: {tickets} tickets, {leads} leads")
```

### Troubleshooting

**Issue: Emails not being routed**

**Check:**
1. Server Script is enabled
2. Email has `email_account` set
3. Sender is not blocked
4. Check error logs for exceptions

**Solution:**
```python
# Check recent communications
comms = frappe.get_all(
    "Communication",
    filters={"creation": [">=", frappe.utils.add_days(frappe.utils.nowdate(), -1)]},
    fields=["name", "sender", "reference_doctype", "reference_name"],
    limit=20
)

for comm in comms:
    print(f"{comm.name}: {comm.sender} → {comm.reference_doctype or 'NOT ROUTED'}")
```

**Issue: Duplicate tickets being created**

**Check:**
1. Reply detection is working
2. In-Reply-To header is present
3. Customer has multiple email addresses

**Solution:**
```python
# Check for duplicates
sender = "customer@example.com"
tickets = frappe.get_all(
    "HD Ticket",
    filters={"raised_by": sender, "status": ["in", ["Open", "Replied"]]},
    fields=["name", "subject", "creation"],
    order_by="creation desc"
)

print(f"Found {len(tickets)} open tickets for {sender}")
```

**Issue: Wrong routing (Support → CRM or vice versa)**

**Check:**
1. Recipient email address
2. Customer history
3. Error logs for routing decision

**Solution:**
```python
# Check customer history
sender = "customer@example.com"

has_tickets = frappe.db.exists("HD Ticket", {"raised_by": sender})
has_leads = frappe.db.exists("CRM Lead", {"email": sender})

print(f"Has Tickets: {has_tickets}, Has Leads: {has_leads}")
```

### Performance Considerations

**Database Queries:**
- Uses indexed fields (`email`, `raised_by`, `message_id`)
- Limits queries to necessary fields only
- Orders by `modified desc` for efficiency

**Optimization Tips:**
1. Add index on `Communication.message_id`
2. Add index on `Communication.in_reply_to`
3. Add index on `HD Ticket.raised_by`
4. Add index on `CRM Lead.email`

**Query Optimization:**
```sql
-- Add indexes (run in bench console)
ALTER TABLE `tabCommunication` ADD INDEX idx_message_id (`message_id`(255));
ALTER TABLE `tabCommunication` ADD INDEX idx_in_reply_to (`in_reply_to`(255));
ALTER TABLE `tabHD Ticket` ADD INDEX idx_raised_by (`raised_by`(255));
ALTER TABLE `tabCRM Lead` ADD INDEX idx_email (`email`(255));
```

### Integration with Other Features

**Relationship with `custom_zip_erp` Python Module:**

The Server Script and `apps/custom_zip_erp/custom_zip_erp/email_routing.py` serve similar purposes but are implemented differently:

| Feature | Server Script | Python Module (`custom_zip_erp`) |
|---------|--------------|----------------------------------|
| **Trigger** | Before Save (DocType Event) | DocType Hook |
| **Location** | ERPNext UI (Server Script) | Python File |
| **Deployment** | No code deployment needed | Requires bench restart |
| **Error Logging** | Comprehensive (every step) | Moderate |
| **Contact Creation** | Built-in function | Separate implementation |
| **Reply Detection** | 3-level (In-Reply-To, References, Recent) | Advanced (includes transferred leads) |
| **Duplicate Prevention** | Via `via_customer_portal = 0` | Via no description |
| **Email Alias Tracking** | Not implemented | `custom_reply_email_alias` field |
| **Transferred Lead Handling** | Not implemented | Advanced (checks alias + sender) |

**Recommendation:** 
- Use **Server Script** for simpler deployments and easier modifications
- Use **Python Module** for advanced features like email alias tracking and transferred lead detection
- Both can coexist (Server Script checks for existing reference)

**Works With:**
- ✅ Email Transfer System (respects transferred emails)
- ✅ Phone Merging (tickets created have proper contacts)
- ✅ Order History (routed tickets can fetch order history)
- ✅ RingCentral Integration (complementary communication channels)

**Coordination:**
- Email routing runs BEFORE transfer
- Transfer sets `reference_doctype`, preventing re-routing
- Contact creation ensures phone fields work properly

### Security Considerations

**Access Control:**
- Uses `ignore_permissions=True` for system operations
- Only processes incoming external emails
- Blocks internal domain emails
- Blocks system/automated emails

**Data Privacy:**
- Logs sanitized data (truncated subjects/senders)
- No sensitive data in error logs
- Email content not logged

**Injection Prevention:**
- No SQL injection risk (uses parameterized queries)
- No code injection (doesn't eval user input)
- Email addresses validated by Frappe framework

### Maintenance

**Weekly Checks:**
1. Review error logs for routing failures
2. Check for blocked legitimate emails
3. Verify routing statistics (tickets vs leads ratio)
4. Monitor duplicate ticket creation

**Monthly Tasks:**
1. Analyze routing patterns
2. Update blocked email list if needed
3. Review and archive old error logs
4. Performance optimization if needed

**Backup:**
```python
# Export server script as JSON
import frappe
script = frappe.get_doc("Server Script", "Communication Before Save")
print(frappe.as_json(script))
```

Save output to version control.

### Future Enhancements

**Potential Improvements:**
1. **AI-Powered Classification**: Use ML to predict ticket vs lead
2. **Customer Intent Detection**: Analyze email content for urgency
3. **Multi-Language Support**: Detect language and route accordingly
4. **Priority Assignment**: Auto-set priority based on keywords
5. **Auto-Assignment**: Route to specific agents based on content
6. **SLA Tracking**: Start SLA timer on ticket creation
7. **Tag Auto-Assignment**: Add tags based on email content
8. **Sentiment Analysis**: Flag negative sentiment emails

---

## Summary

This documentation covers all major enhancements made to the CRM and Helpdesk modules:

**CRM (5 Features):**
1. ✅ Sales Order validation on Deals
2. ✅ Lead auto-merge by phone
3. ✅ Call transcription viewing
4. ✅ Email transfer to Helpdesk
5. ✅ Graceful payload processing

**Helpdesk (9 Features):**
1. ✅ Click-to-call functionality
2. ✅ Call recording playback
3. ✅ Call transcription viewing
4. ✅ Automatic ticket phone merging
5. ✅ Phone number save fix
6. ✅ Recording URL extraction
7. ✅ Contact mobile field fix
8. ✅ Order history display
9. ✅ Email transfer to CRM

**Email Routing:**
- ✅ Intelligent email routing (Support vs Sales)
- ✅ Reply thread detection (3-level)
- ✅ Contact pre-creation
- ✅ Duplicate prevention
- ✅ Comprehensive error handling

**Integration:**
- ✅ RingCentral OAuth authentication with auto-refresh
- ✅ ERPNext Sales Order integration
- ✅ Production status tracking
- ✅ Pabbly webhook processing
- ✅ Email routing system (Server Script)

All features are production-ready and fully tested.

---
**Documented By:** Harshit Verma
**Document Version:** 1.0  
**Last Updated:** January 2, 2026  
**Status:** Complete

---

**For Support:**
- Review this documentation
- Check Error Logs in Frappe
- Review existing documentation files in repository
- Check RingCentral API status

