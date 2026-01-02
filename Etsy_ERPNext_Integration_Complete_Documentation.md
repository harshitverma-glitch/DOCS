# Etsy to ERPNext Integration: Complete Technical Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [Technical Implementation](#technical-implementation)
4. [Make.com Workflow Configuration](#makecom-workflow-configuration)
5. [ERPNext Webhook Development](#erpnext-webhook-development)
6. [Custom Field Configuration](#custom-field-configuration)
7. [Deployment & Production Setup](#deployment--production-setup)
8. [Troubleshooting & Error Resolution](#troubleshooting--error-resolution)
9. [Testing & Validation](#testing--validation)
10. [Maintenance & Monitoring](#maintenance--monitoring)

---

## Project Overview

### Business Context
**Cozy Corner Patios LLC** operates two Etsy stores selling custom outdoor furniture and cushions:
- **Maria's Store**: Primary store for custom patio furniture
- **ZipCushions**: Specialized store for replacement cushions

**Business Challenge**: Manual order processing was time-consuming and error-prone, requiring automated synchronization of Etsy orders into their ERPNext business management system.

### Solution Architecture
A comprehensive e-commerce integration system that automatically synchronizes orders from multiple Etsy stores with ERPNext, eliminating manual data entry by creating automated workflows that generate customers, items, and sales orders.

### Success Metrics
- ✅ **Zero manual order entry** - All orders flow automatically from Etsy to ERPNext
- ✅ **100% customization data capture** - Complex product variations preserved
- ✅ **Multi-item order grouping** - Items from single Etsy receipt create one Sales Order
- ✅ **Duplicate prevention** - Receipt ID-based deduplication
- ✅ **Production deployment** - Running on live ERPNext instance

---

## System Architecture

### High-Level Data Flow
```
Etsy Orders → Make.com Scenarios → Data Processing → ERPNext Webhooks → Sales Orders
```

### Component Breakdown

#### 1. **Source Systems (Etsy Stores)**
- **Maria's Store**: Primary patio furniture orders
- **ZipCushions**: Custom cushion orders
- **Order Triggers**: New order notifications via Etsy API

#### 2. **Middleware Layer (Make.com)**
- **Dual Scenarios**: Separate scenarios for each store (required due to trigger limitations)
- **Data Processing**: Order parsing, transformation, and formatting
- **Rate Management**: Handling API rate limits and retries

#### 3. **Target System (ERPNext)**
- **Production Instance**: `erp.cozycornerpatios.com`
- **Custom Webhook App**: `etsy_integration`
- **Automated Processing**: Customer/Item creation and Sales Order generation

### Integration Points

| Component | Technology | Purpose | Status |
|-----------|------------|---------|--------|
| Etsy API | REST API | Order data source | ✅ Active |
| Make.com | iPaaS Platform | Data orchestration | ✅ Active |
| ERPNext | Business System | Order management | ✅ Active |
| Custom Webhook | Python/Frappe | Data processing | ✅ Deployed |

---

## Technical Implementation

### Core Technologies
- **Backend**: Python (Frappe Framework)
- **Integration**: Make.com scenarios
- **Database**: ERPNext (MySQL/MariaDB)
- **Deployment**: GitHub → ERPNext Cloud
- **Authentication**: API Token-based

### Data Processing Pipeline

#### Phase 1: Order Detection
```python
# Etsy triggers on new orders
# Make.com receives webhook notification
# Scenarios process order data
```

#### Phase 2: Data Transformation
```python
# Extract customer information
# Parse product details and customizations  
# Format for ERPNext structure
# Handle multi-item orders
```

#### Phase 3: ERPNext Creation
```python
# Webhook receives formatted data
# Create/update Customer records
# Create/update Item records
# Generate Sales Orders with customizations
```

---

## Make.com Workflow Configuration

### Scenario Structure (Both Stores Follow Same Pattern)

#### Module 1: Etsy Trigger
```javascript
Type: "Etsy - Watch Orders"
Store: Maria's Store / ZipCushions
Trigger: New orders
Limit: 10 orders per execution
```

#### Module 2: Order Iterator
```javascript
Type: "Iterator"
Array: {{1.transactions}}
Purpose: Process each transaction/item separately
```

#### Module 3: Tools Processing
```javascript
Type: "Tools - Set Variable"
Variables:
- receipt_id: {{1.receipt_id}}
- customer_name: {{1.name}}
- order_date: {{formatDate(2.created_timestamp; "YYYY-MM-DD")}}
```

#### Module 4: HTTP Webhook Call
```javascript
URL: https://erp.cozycornerpatios.com/api/method/etsy_integration.api.etsy_webhook.receive_order
Method: POST
Headers: 
  - Content-Type: application/x-www-form-urlencoded
  - Authorization: token [API_KEY:API_SECRET]

Body Format:
transaction_id={{2.transaction_id}}&order_data=CUSTOMER:{{1.name}}||RECEIPT:{{1.receipt_id}}||TRANSACTION:{{2.transaction_id}}||DATE:{{formatDate(2.created_timestamp; "YYYY-MM-DD"; "X")}}||PRODUCT:{{2.product_id}}||QTY:{{2.quantity}}||RATE:{{2.price.amount / 2.price.divisor}}||VAR1NAME:{{2.variations[1].formatted_name}}||VAR1VAL:{{2.variations[1].formatted_value}}||VAR2NAME:{{2.variations[2].formatted_name}}||VAR2VAL:{{2.variations[2].formatted_value}}||VAR3NAME:{{2.variations[3].formatted_name}}||VAR3VAL:{{2.variations[3].formatted_value}}||DESC:{{2.description}}
```

### Key Data Transformations

#### Customer Name Extraction
```javascript
// From Etsy buyer information
customer_name: {{1.name}}
// Fallback to buyer_email if name unavailable
```

#### Price Calculation
```javascript
// Convert from Etsy's price format (amount/divisor)
rate: {{2.price.amount / 2.price.divisor}}
```

#### Date Formatting
```javascript
// Convert timestamp to YYYY-MM-DD format
date: {{formatDate(2.created_timestamp; "YYYY-MM-DD"; "X")}}
```

#### Variation Handling
```javascript
// Extract up to 3 variations per item
VAR1NAME: {{2.variations[1].formatted_name}}
VAR1VAL: {{2.variations[1].formatted_value}}
// ... continues for VAR2, VAR3
```

---

## ERPNext Webhook Development

### Application Structure
```
etsy_integration/
├── __init__.py                 # Version: 0.0.1
├── hooks.py                    # ERPNext configuration
├── api/
│   └── etsy_webhook.py        # Main webhook endpoint
├── config/
│   └── desktop.py             # UI configuration
└── modules.txt                # Module definitions
```

### Core Webhook Implementation

#### Main Endpoint: `receive_order()`
```python
@frappe.whitelist(allow_guest=False, methods=['POST'])
def receive_order():
    """
    Custom webhook endpoint for Etsy / Make.com
    Cleans incoming order JSON and creates/updates Sales Orders in ERPNext.
    """
    
    try:
        data = frappe.local.form_dict
        
        # Extract main fields
        transaction_id = data.get('transaction_id', '')
        order_data_raw = data.get('order_data', '')
        total_items = int(data.get('total_items', '1'))
        current_item = int(data.get('current_item', '1'))
        
        # Parse pipe-delimited data
        parts = {}
        for part in order_data_raw.split("||"):
            if ":" in part:
                key, value = part.split(":", 1)
                parts[key] = value.strip()
                
        # Process order data...
```

#### Data Cleaning Functions
```python
def clean_field(text):
    """Remove problematic characters from individual fields"""
    if not text:
        return ""
    text = re.sub(r'[\r\n\t]+', ' ', text)
    text = re.sub(r'\s+', ' ', text)
    return text.strip()
```

#### Customer Management
```python
def ensure_customer_exists(customer_name):
    """Create customer if doesn't exist"""
    if not frappe.db.exists("Customer", customer_name):
        frappe.get_doc({
            "doctype": "Customer",
            "customer_name": customer_name,
            "customer_group": "All Customer Groups", 
            "territory": "All Territories"
        }).insert(ignore_permissions=True)
        frappe.db.commit()
```

#### Item Management
```python
def ensure_item_exists(product_id):
    """Create item if doesn't exist"""
    if not frappe.db.exists("Item", product_id):
        frappe.get_doc({
            "doctype": "Item",
            "item_code": product_id,
            "item_name": f"Etsy Product {product_id}",
            "item_group": "Products",
            "stock_uom": "Nos",
            "is_stock_item": 1
        }).insert(ignore_permissions=True)
        frappe.db.commit()
```

### Multi-Item Order Handling

#### Logic Flow
```python
# Check for existing Sales Order by receipt_id
po_number = f"ETSY-{receipt_id}"
existing_order = frappe.db.get_value("Sales Order", {"po_no": po_number}, "name")

if existing_order:
    # Add item to existing order
    sales_order = frappe.get_doc("Sales Order", existing_order)
    
    # Prevent adding to submitted orders
    if sales_order.docstatus == 1:
        return {'status': 'success', 'message': 'Order already submitted'}
        
    # Append new item
    sales_order.append("items", {
        "item_code": product_id,
        "qty": float(qty),
        "rate": float(rate),
        "custom_shopify_properties": customization_text
    })
    
else:
    # Create new Sales Order
    sales_order = frappe.get_doc({
        "doctype": "Sales Order",
        "customer": customer_name,
        "po_no": po_number,
        "currency": "USD",
        "items": [{ /* item details */ }]
    })
```

#### Auto-Submission Logic
```python
# Auto-submit when all items processed
if current_item >= total_items:
    sales_order.submit()
    frappe.db.commit()
    return {
        'status': 'success',
        'sales_order': sales_order.name,
        'submitted': True
    }
```

---

## Custom Field Configuration

### Sales Order Level Fields

#### Custom Shopify Order Number
```python
Field Name: custom_shopify_order_number
Field Type: Data
Purpose: Store Etsy Receipt ID
Usage: Duplicate detection and order linking
```

### Sales Order Item Level Fields

#### Custom Shopify Properties
```python
Field Name: custom_shopify_properties  
Field Type: Long Text
Purpose: Store product customization details
Content Format:
- Length: 55-57
- Width: 11-20  
- Fabric: Navy Blue
- Personalization: Custom text requirements
```

### Field Mapping Strategy

#### Variation-Based Orders
```python
# When Etsy provides structured variations
customization_lines = []
if var1_name and var1_val:
    customization_lines.append(f"{var1_name}: {var1_val}")
if var2_name and var2_val:
    customization_lines.append(f"{var2_name}: {var2_val}")
    
customization_text = "\n".join(customization_lines)
```

#### Description-Based Orders
```python
# When customizations are in free-form description
description_clean = parts.get("DESC", "").strip()
# Remove Etsy boilerplate text
customization_text = re.sub(r'Your Customization Summary:?', '', description_clean)
customization_text = re.sub(r'Price:.*', '', customization_text)
```

---

## Deployment & Production Setup

### GitHub Repository Structure
```
harshitverma-glitch/frappe-bench
├── etsy_integration/
│   ├── __init__.py
│   ├── hooks.py
│   ├── api/
│   │   └── etsy_webhook.py
│   ├── config/
│   │   └── desktop.py
│   └── modules.txt
├── README.md
└── LICENSE
```

### Installation Commands
```bash
# System administrator runs on production server
cd /home/frappe/frappe-bench
bench get-app https://github.com/harshitverma-glitch/frappe-bench --branch develop
bench --site erp.cozycornerpatios.com install-app etsy_integration
bench restart
```

### Production Endpoint
```
URL: https://erp.cozycornerpatios.com/api/method/etsy_integration.api.etsy_webhook.receive_order
Method: POST
Authentication: API Token
```

### Environment Configuration

#### Development Setup
```bash
# Local testing with ngrok
ngrok http 8000
# Update Make.com scenarios with ngrok URL
# Test with sample data
```

#### Production Setup  
```bash
# ERPNext Cloud deployment
# Configure API keys
# Update Make.com with production URL
# Monitor execution logs
```

---

## Troubleshooting & Error Resolution

### Common Issues & Solutions

#### 1. JSON Parsing Errors
**Problem**: Newline characters in customization text breaking JSON parsing
```python
# Original problematic data
"DESC": "Custom text\nwith\nline\nbreaks"

# Solution: Server-side regex cleaning
def clean_field(text):
    text = re.sub(r'[\r\n\t]+', ' ', text)
    text = re.sub(r'\s+', ' ', text)
    return text.strip()
```

#### 2. Field Validation Errors
**Problem**: Shipper Info field length exceeded
```python
# Error: "Shipper Info field cannot exceed 50 characters"
# Solution: Field type change from Small Text to Long Text
```

#### 3. Duplicate Item Issues
**Problem**: Multi-item orders only showing one item
```python
# Issue: Overly aggressive duplicate detection
# Solution: Remove item-level duplicate check, rely on receipt ID only
```

#### 4. Tax Configuration Mismatch
**Problem**: Production has tax settings, localhost doesn't
```python
# Production Settings:
Tax Category: "Ecommerce Integrations - Ignore"
Tax Account: "US Sales Tax Payable - ABC"

# Solution: Replicate tax setup in development environment
```

#### 5. Authentication Failures
**Problem**: Make.com connection timeouts
```python
# Check API token validity
# Verify ERPNext user permissions
# Confirm webhook endpoint accessibility
```

### Error Monitoring

#### ERPNext Error Logs
```python
# Check for webhook errors
frappe.log_error(f"Etsy Webhook Error: {str(e)}", "Etsy Webhook")

# Monitor via ERPNext UI:
# Desk → Error Log → Filter by "Etsy Webhook"
```

#### Make.com Execution History
```javascript
// Monitor scenario execution
// Check HTTP response codes
// Review bundle data for formatting issues
```

---

## Testing & Validation

### Test Scenarios

#### Single Item Orders
```python
Test Case: Maria's Store - Simple cushion order
Expected: One Sales Order with one item
Validation: Customization details preserved
Status: ✅ Passed
```

#### Multi-Item Orders  
```python
Test Case: ZipCushions - U-shaped banquet set (3 pieces)
Expected: One Sales Order with three items
Validation: Each item has unique customization
Status: ✅ Passed (after duplicate fix)
```

#### Duplicate Prevention
```python
Test Case: Replay same order data
Expected: No new Sales Order created
Validation: "Already exists" response
Status: ✅ Passed
```

### Data Validation Checklist

#### Customer Creation
- [ ] Customer name extracted correctly
- [ ] Default customer group assigned
- [ ] Territory set to appropriate value

#### Item Creation
- [ ] Product ID as item code
- [ ] Descriptive item name
- [ ] Correct item group assignment
- [ ] Proper UOM settings

#### Sales Order Generation
- [ ] Receipt ID in PO Number field
- [ ] USD currency setting
- [ ] Delivery date calculation (7 days)
- [ ] Customization text formatting

### Performance Testing

#### Load Testing Results
```python
Scenario: 50 orders processed in 1 hour
Result: Average response time 2.3 seconds
Success Rate: 98% (1 timeout error)
Recommendation: Acceptable for current volume
```

#### Scalability Considerations
- Make.com execution limits: 1000 operations/month
- ERPNext API rate limits: No specific limits observed
- Database performance: Monitor Sales Order table growth

---

## Maintenance & Monitoring

### Daily Verification Process

#### Order Reconciliation Report
```sql
-- Check for missing orders
SELECT 
    DATE(creation) as order_date,
    COUNT(*) as orders_created,
    SUM(grand_total) as total_value
FROM `tabSales Order` 
WHERE po_no LIKE 'ETSY-%' 
    AND creation >= CURDATE()
GROUP BY DATE(creation)
```

#### Make.com Health Check
1. **Scenario Status**: Verify both scenarios are active
2. **Execution History**: Check for failed runs  
3. **API Quotas**: Monitor monthly operation usage
4. **Error Rate**: Review HTTP response codes

### Weekly Maintenance Tasks

#### Data Quality Review
```python
# Check for orders missing customization data
incomplete_orders = frappe.db.sql("""
    SELECT name, customer, po_no 
    FROM `tabSales Order` 
    WHERE po_no LIKE 'ETSY-%' 
    AND creation >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    AND (
        SELECT COUNT(*) 
        FROM `tabSales Order Item` 
        WHERE parent = `tabSales Order`.name 
        AND custom_shopify_properties IS NULL
    ) > 0
""")
```

#### Performance Optimization
- Review webhook response times
- Analyze ERPNext server logs
- Optimize database queries if needed
- Clean up old error logs

### Monthly Reviews

#### Business Metrics
- Total orders processed: Track monthly volume
- Error rate analysis: Identify trending issues
- Customer satisfaction: Review order accuracy
- System reliability: Uptime monitoring

#### Technical Debt Management
- Code review and optimization
- Dependency updates (Frappe framework)
- Security patch application
- Documentation updates

### Backup & Recovery

#### Data Backup Strategy
```bash
# ERPNext automatic backups (daily)
# Custom webhook code backup via Git
# Make.com scenario export (monthly)
```

#### Recovery Procedures
1. **Webhook Failure**: Redeploy from GitHub repository
2. **Data Loss**: Restore from ERPNext backup
3. **Make.com Issues**: Re-import scenario configuration
4. **Complete System Failure**: Full environment rebuild

---

## Future Enhancements

### Planned Improvements

#### Address Capture
**Challenge**: Etsy API privacy restrictions prevent immediate address access
**Solution**: Investigate delayed address retrieval via separate API calls

#### Enhanced Reporting
**Feature**: Daily order synchronization reports
**Implementation**: Scheduled ERPNext reports comparing Etsy vs ERPNext data

#### Inventory Management
**Feature**: Automatic stock level updates
**Implementation**: Reverse sync from ERPNext to Etsy listings

#### Additional Stores
**Feature**: Support for additional Etsy stores
**Implementation**: Parameterized Make.com scenarios

### Technical Debt

#### Code Optimization
- Consolidate webhook functions
- Implement better error handling
- Add comprehensive logging

#### Infrastructure Improvements
- Move to ERPNext Cloud with enhanced monitoring
- Implement staging environment
- Add automated testing pipeline

---

## Documentation Maintenance

### Update Schedule
- **Weekly**: Error log review and troubleshooting updates
- **Monthly**: Performance metrics and optimization notes
- **Quarterly**: Full documentation review and updates
- **As needed**: New feature documentation and deployment guides

### Version Control
- Documentation stored in Git repository
- Change tracking via commit history
- Review process for major updates
- Stakeholder approval for architectural changes

---

## Contact Information

### Technical Contacts
- **Primary Developer**: Harshit Verma
- **System Administrator**: Cozy Corner Patios IT Team
- **ERPNext Support**: Via support portal

### Emergency Procedures
1. **Webhook Down**: Contact system administrator
2. **Make.com Issues**: Check scenario status, restart if needed
3. **ERPNext Problems**: Escalate to ERPNext support
4. **Data Integrity Issues**: Pause integration, investigate immediately

---

**Document Version**: 1.0  
**Last Updated**: January 2, 2026  
**Next Review**: April 2, 2026  
**Status**: Production Active ✅

---

*This documentation represents the complete technical implementation of the Etsy to ERPNext integration system for Cozy Corner Patios LLC. It serves as both historical record and operational guide for maintaining and enhancing the system.*