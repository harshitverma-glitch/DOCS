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
7. [Deployment Guide](#deployment-guide)
8. [API Reference](#api-reference)
9. [Troubleshooting](#troubleshooting)

---

## Executive Summary

This document provides comprehensive technical documentation for all enhancements made to the CRM and Helpdesk modules, including RingCentral telephony integration, email transfer capabilities, and order history tracking from ERPNext.

### Key Statistics

- **Total Features Implemented:** 14
- **CRM Features:** 5
- **Helpdesk Features:** 9
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

### Configuration

**RingCentral Settings:**
1. Go to CRM → Settings → RingCentral Settings
2. Enter Client ID and Client Secret
3. Click "Authorize" to complete OAuth flow
4. Verify tokens are stored

**Pabbly Webhooks:**
1. Configure webhook URL: `https://yourdomain.com/api/method/helpdesk.api.ringcentral_webhook.pabbly_webhook`
2. Set webhook events: call_completed, recording_available
3. Test webhook delivery

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

**Integration:**
- ✅ RingCentral OAuth authentication with auto-refresh
- ✅ ERPNext Sales Order integration
- ✅ Production status tracking
- ✅ Pabbly webhook processing

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

