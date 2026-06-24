# SQL Assignment 2

Below is a structured list of queries for **Mixed Party + Order** scenarios and **Inventory Management & Transfers**, along with several additional questions. Each section contains the **Business Problem** and the **Fields to Retrieve**, **without** example SQL code.

---

## 5. Mixed Party + Order Queries

### 5.1 Shipping Addresses for October 2023 Orders

**Business Problem:**  
Customer Service might need to verify addresses for orders placed or completed in October 2023. This helps ensure shipments are delivered correctly and prevents address-related issues.

**Fields to Retrieve:**  
- `ORDER_ID` 
- `PARTY_ID` (Customer ID)  
- `CUSTOMER_NAME` (or FIRST_NAME / LAST_NAME)  
- `STREET_ADDRESS`  
- `CITY` 
- `STATE_PROVINCE`
- `POSTAL_CODE`  
- `COUNTRY_CODE`  
- `ORDER_STATUS`  
- `ORDER_DATE`
##QUERY
```sql
SELECT 
    oh.order_id, 
    orole.party_id, 
    CONCAT(per.first_name, ' ', per.last_name) AS customer_name, 
    pa.address1 AS street_address, 
    pa.city, 
    pa.state_province_geo_id AS state_province, 
    pa.postal_code, 
    pa.country_geo_id AS country_code, 
    oh.status_id AS order_status, 
    oh.order_date
FROM order_header oh
JOIN order_role orole ON oh.order_id = orole.order_id AND orole.role_type_id = 'PLACING_CUSTOMER'
LEFT JOIN person per ON orole.party_id = per.party_id
JOIN order_contact_mech ocm ON oh.order_id = ocm.order_id AND ocm.contact_mech_purpose_type_id = 'SHIPPING_LOCATION'
JOIN postal_address pa ON ocm.contact_mech_id = pa.contact_mech_id
WHERE oh.order_date >= '2023-10-01' AND oh.order_date < '2023-11-01';

```


### 5.2 Orders from New York

**Business Problem:**  
Companies often want region-specific analysis to plan local marketing, staffing, or promotions in certain areas—here, specifically, New York.

**Fields to Retrieve:**  
- `ORDER_ID` 
- `CUSTOMER_NAME` 
- `STREET_ADDRESS` (or shipping address detail)  
- `CITY`  
- `STATE_PROVINCE`
- `POSTAL_CODE` 
- `TOTAL_AMOUNT`
- `ORDER_DATE`  
- `ORDER_STATUS`
##QUERY
```sql
SELECT 
    oh.order_id, 
    CONCAT(per.first_name, ' ', per.last_name) AS customer_name, 
    pa.address1 AS street_address, 
    pa.city, 
    pa.state_province_geo_id AS state_province, 
    pa.postal_code, 
    oh.grand_total AS total_amount, 
    oh.order_date, 
    oh.status_id AS order_status
FROM order_header oh
JOIN order_role orole ON oh.order_id = orole.order_id AND orole.role_type_id = 'PLACING_CUSTOMER'
LEFT JOIN person per ON orole.party_id = per.party_id
JOIN order_contact_mech ocm ON oh.order_id = ocm.order_id AND ocm.contact_mech_purpose_type_id = 'SHIPPING_LOCATION'
JOIN postal_address pa ON ocm.contact_mech_id = pa.contact_mech_id
WHERE pa.state_province_geo_id IN ('NY', 'USA_NY') OR pa.city = 'New York';
```

---

### 5.3 Top-Selling Product in New York

**Business Problem:**  
Merchandising teams need to identify the best-selling product(s) in a specific region (New York) for targeted restocking or promotions.

**Fields to Retrieve:**  
- `PRODUCT_ID`  
- `INTERNAL_NAME`
- `TOTAL_QUANTITY_SOLD`  
- `CITY` / `STATE` (within New York region) 
- `REVENUE` (optionally, total sales amount)
  ##QUERY
```sql
SELECT 
    oi.product_id, 
    p.internal_name, 
    SUM(oi.quantity) AS total_quantity_sold, 
    pa.city, 
    pa.state_province_geo_id AS state, 
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_header oh
JOIN order_item oi ON oh.order_id = oi.order_id
JOIN product p ON oi.product_id = p.product_id
JOIN order_contact_mech ocm ON oh.order_id = ocm.order_id AND ocm.contact_mech_purpose_type_id = 'SHIPPING_LOCATION'
JOIN postal_address pa ON ocm.contact_mech_id = pa.contact_mech_id
WHERE pa.state_province_geo_id IN ('NY', 'USA_NY') OR pa.city = 'New York'
GROUP BY oi.product_id, p.internal_name, pa.city, pa.state_province_geo_id
ORDER BY total_quantity_sold DESC
LIMIT 1;

```

### 7.3 Store-Specific (Facility-Wise) Revenue

**Business Problem:**  
Different physical or online stores (facilities) may have varying levels of performance. The business wants to compare revenue across facilities for sales planning and budgeting.

**Fields to Retrieve:**  
- `FACILITY_ID`
- `FACILITY_NAME`  
- `TOTAL_ORDERS` 
- `TOTAL_REVENUE`  
- `DATE_RANGE`
  ##QUERY
  
```sql
SELECT 
    f.facility_id, 
    f.facility_name, 
    COUNT(DISTINCT oh.order_id) AS total_orders, 
    SUM(oi.quantity * oi.unit_price) AS total_revenue, 
    MIN(oh.order_date) AS date_range_start,
    MAX(oh.order_date) AS date_range_end
FROM order_header oh
JOIN order_item_ship_group oisg ON oh.order_id = oisg.order_id
JOIN facility f ON oisg.facility_id = f.facility_id
JOIN order_item oi ON oh.order_id = oi.order_id AND oisg.ship_group_seq_id = oi.ship_group_seq_id
WHERE oh.status_id = 'ORDER_COMPLETED'
GROUP BY f.facility_id, f.facility_name;

```

## 8. Inventory Management & Transfers

### 8.1 Lost and Damaged Inventory

**Business Problem:**  
Warehouse managers need to track “shrinkage” such as lost or damaged inventory to reconcile physical vs. system counts.

**Fields to Retrieve:**  
- `INVENTORY_ITEM_ID` 
- `PRODUCT_ID` 
- `FACILITY_ID` 
- `QUANTITY_LOST_OR_DAMAGED` 
- `REASON_CODE` (Lost, Damaged, Expired, etc.)  
- `TRANSACTION_DATE`
  ##QUERY
```sql
SELECT 
    iiv.inventory_item_id, 
    ii.product_id, 
    ii.facility_id, 
    ABS(iiv.quantity_on_hand_var) AS quantity_lost_or_damaged, 
    vr.description AS reason_code, 
    iiv.variance_date AS transaction_date
FROM inventory_item_variance iiv
JOIN inventory_item ii ON iiv.inventory_item_id = ii.inventory_item_id
JOIN variance_reason vr ON iiv.variance_reason_id = vr.variance_reason_id
WHERE iiv.variance_reason_id IN ('VAR_LOST', 'VAR_DAMAGED', 'VAR_STOLEN')
   OR vr.description LIKE '%Lost%' 
   OR vr.description LIKE '%Damaged%';

```

### 8.2 Low Stock or Out of Stock Items Report

**Business Problem:**  
Avoiding out-of-stock situations is critical. This report flags items that have fallen below a certain reorder threshold or have zero available stock.

**Fields to Retrieve:**  
- `PRODUCT_ID`
- `PRODUCT_NAME` 
- `FACILITY_ID`  
- `QOH` (Quantity on Hand)  
- `ATP` (Available to Promise)  
- `REORDER_THRESHOLD` 
- `DATE_CHECKED`
- ##QUERY
```sql
SELECT 
    ii.product_id, 
    p.internal_name AS product_name, 
    ii.facility_id, 
    SUM(ii.quantity_on_hand_total) AS qoh, 
    SUM(ii.available_to_promise_total) AS atp, 
    pf.minimum_stock AS reorder_threshold, 
    CURRENT_DATE AS date_checked
FROM inventory_item ii
JOIN product p ON ii.product_id = p.product_id
LEFT JOIN product_facility pf ON ii.product_id = pf.product_id AND ii.facility_id = pf.facility_id
GROUP BY ii.product_id, p.internal_name, ii.facility_id, pf.minimum_stock
HAVING SUM(ii.available_to_promise_total) <= COALESCE(pf.minimum_stock, 0);

```

### 8.3 Retrieve the Current Facility (Physical or Virtual) of Open Orders

**Business Problem:**  
The business wants to know where open orders are currently assigned, whether in a physical store or a virtual facility (e.g., a distribution center or online fulfillment location).

**Fields to Retrieve:**  
- `ORDER_ID`  
- `ORDER_STATUS`
- `FACILITY_ID`  
- `FACILITY_NAME`  
- `FACILITY_TYPE_ID`
- ##QUERY
```sql
SELECT 
    oh.order_id, 
    oh.status_id AS order_status, 
    f.facility_id, 
    f.facility_name, 
    f.facility_type_id
FROM order_header oh
JOIN order_item_ship_group oisg ON oh.order_id = oisg.order_id
JOIN facility f ON oisg.facility_id = f.facility_id
WHERE oh.status_id IN ('ORDER_CREATED', 'ORDER_APPROVED');

```

### 8.4 Items Where QOH and ATP Differ

**Business Problem:**  
Sometimes the **Quantity on Hand (QOH)** doesn’t match the **Available to Promise (ATP)** due to pending orders, reservations, or data discrepancies. This needs review for accurate fulfillment planning.

**Fields to Retrieve:**  
- `PRODUCT_ID`
- `FACILITY_ID`
- `QOH` (Quantity on Hand)  
- `ATP` (Available to Promise)  
- `DIFFERENCE` (QOH - ATP)
- ##QUERY
```sql
SELECT 
    ii.product_id, 
    ii.facility_id, 
    SUM(ii.quantity_on_hand_total) AS qoh, 
    SUM(ii.available_to_promise_total) AS atp, 
    SUM(ii.quantity_on_hand_total) - SUM(ii.available_to_promise_total) AS difference
FROM inventory_item ii
GROUP BY ii.product_id, ii.facility_id
HAVING SUM(ii.quantity_on_hand_total) <> SUM(ii.available_to_promise_total);

```

### 8.5 Order Item Current Status Changed Date-Time

**Business Problem:**  
Operations teams need to audit when an order item’s status (e.g., from “Pending” to “Shipped”) was last changed, for shipment tracking or dispute resolution.

**Fields to Retrieve:**  
- `ORDER_ID` 
- `ORDER_ITEM_SEQ_ID` 
- `CURRENT_STATUS_ID` 
- `STATUS_CHANGE_DATETIME`
- `CHANGED_BY`
- ##QUERY
```sql
SELECT 
    oi.order_id, 
    oi.order_item_seq_id, 
    oi.status_id AS current_status_id, 
    os.status_datetime AS status_change_datetime, 
    os.status_user_login AS changed_by
FROM order_item oi
JOIN order_status os ON oi.order_id = os.order_id AND oi.order_item_seq_id = os.order_item_seq_id AND oi.status_id = os.status_id
WHERE os.status_datetime = (
    SELECT MAX(status_datetime) 
    FROM order_status os2 
    WHERE os2.order_id = oi.order_id AND os2.order_item_seq_id = oi.order_item_seq_id AND os2.status_id = oi.status_id
);

```


### 8.6 Total Orders by Sales Channel

**Business Problem:**  
Marketing and sales teams want to see how many orders come from each channel (e.g., web, mobile app, in-store POS, marketplace) to allocate resources effectively.

**Fields to Retrieve:**  
- `SALES_CHANNEL`
- `TOTAL_ORDERS`
- `TOTAL_REVENUE`
- `REPORTING_PERIOD`
  ##QUERY
```sql
SELECT 
    oh.sales_channel_enum_id AS sales_channel, 
    COUNT(DISTINCT oh.order_id) AS total_orders, 
    SUM(oh.grand_total) AS total_revenue, 
    'CURRENT_YEAR' AS reporting_period 
FROM order_header oh
WHERE oh.order_date >= '2024-01-01'
GROUP BY oh.sales_channel_enum_id;

```
