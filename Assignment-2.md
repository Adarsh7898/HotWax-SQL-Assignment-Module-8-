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

```sql
SELECT 
oh.ORDER_ID,
role.PARTY_ID,
CONCAT(p.first_name,p.last_name) AS CUSTOMER_NAME,
COALESCE(pa.address1,pa.address2) AS STREET_ADDRESS,
pa.CITY,
pa.state_province_geo_id AS STATE_PROVINCE,
pa.POSTAL_CODE,
pa.country_geo_id AS COUNTRY_CODE,
oh.status_id AS ORDER_STATUS,
oh.ORDER_DATE

FROM order_header oh
JOIN order_role role USING(order_id)
JOIN person p USING(party_id)
JOIN order_contact_mech ocm USING(order_id)
JOIN contact_mech cm USING(contact_mech_id)
JOIN postal_address pa USING(contact_mech_id)
WHERE oh.order_date >= '2023-10-01' AND oh.order_date < '2023-11-01'



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


```sql
SELECT
oh.ORDER_ID,
CONCAT(p.first_name,' ',p.last_name) AS CUSTOMER_NAME,
COALESCE(address1,address2) AS STREET_ADDRESS ,
pa.CITY,
pa.STATE_PROVINCE_GEO_ID,
pa.POSTAL_CODE,
oh.grand_total AS TOTAL_AMOUNT,
oh.ORDER_DATE,
oh.status_id AS ORDER_STATUS

FROM order_header oh
LEFT JOIN order_role role USING(order_id)
JOIN person p USING(party_id)
JOIN order_contact_mech ocm USING(order_id)
JOIN postal_address pa USING(contact_mech_id)
WHERE pa.city = 'NEW YORK'
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

```sql
SELECT 
oi.PRODUCT_ID,
p.INTERNAL_NAME,
SUM(oi.quantity) AS TOTAL_QUANTITY_SOLD,
pa.CITY,
pa.STATE_PROVINCE_GEO_ID,
SUM(oi.unit_price * oi.quantity) AS REVENUE

FROM order_item oi
JOIN product p USING(product_id)
JOIN order_contact_mech ocm USING(order_id)
JOIN contact_mech cm USING(contact_mech_id)
JOIN postal_address pa USING(contact_mech_id)
WHERE pa.state_province_geo_id IN ('NY', 'USA_NY') OR pa.city = 'New York'
AND ocm.contact_mech_purpose_type_id='SHIPPING_LOCATION'
GROUP BY oi.product_id, p.internal_name, pa.city, pa.state_province_geo_id
ORDER BY TOTAL_QUANTITY_SOLD DESC
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

  
```sql
SELECT
f.FACILITY_ID,
f.FACILITY_NAME,
SUM(DISTINCT oh.order_id) AS TOTAL_ORDERS,
CONCAT(SUM(oi.unit_price * oi.quantity),'$' )AS TOTAL_REVENUE,
CONCAT(MIN(oh.order_date),' TO ', MAX(order_date)) AS DATE_RANGE

FROM order_header oh
JOIN order_item oi USING(order_id) 
JOIN order_item_ship_group oisg USING(order_id)
JOIN facility f USING(facility_id)
WHERE oh.status_id = 'ORDER_COMPLETED'
GROUP BY f.facility_id, f.facility_name 


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

```sql
SELECT 
iiv.inventory_item_id, 
ii.product_id, 
ii.facility_id, 
iiv.quantity_on_hand_var AS quantity_lost_or_damaged, 
vr.description AS reason_code, 
iiv.created_stamp AS transaction_date
FROM inventory_item_variance iiv
JOIN inventory_item ii ON iiv.inventory_item_id = ii.inventory_item_id
JOIN variance_reason vr ON iiv.variance_reason_id = vr.variance_reason_id
WHERE iiv.variance_reason_id IN ('VAR_LOST', 'VAR_DAMAGED', 'VAR_STOLEN')


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

```sql
SELECT 
p.PRODUCT_ID,
p.PRODUCT_NAME,
pf.FACILITY_ID,
SUM(ii.quantity_on_hand_total) AS QOH,
SUM(ii.available_to_promise_total) AS ATP,
pf.minimum_stock AS REORDER_THRESHOLD,
CURRENT_DATE AS DATE_CHECKED

FROM inventory_item ii
JOIN product_facility pf USING(facility_id)
JOIN product p ON p.product_id = ii.product_id
GROUP BY ii.product_id,ii.facility_id
HAVING SUM(available_to_promise_total) <=0

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

```sql
SELECT
oh.ORDER_ID,
oh.status_id AS ORDER_STATUS,
oisg.FACILITY_ID,
f.FACILITY_NAME,
f.FACILITY_TYPE_ID

FROM order_header oh
JOIN order_item_ship_group oisg USING(order_id)
JOIN facility f USING(facility_id)
WHERE oh.status_id IN('ORDER_APPROVED','ORDER_CREATED')

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

```sql
SELECT 
PRODUCT_ID,
FACILITY_ID,
SUM(quantity_on_hand_total) AS QOH,
SUM(available_to_promise_total) AS ATP,
SUM(quantity_on_hand_total) - SUM(available_to_promise_total) AS DIFFERENCE

FROM inventory_item 
GROUP BY product_id, facility_id
HAVING SUM(quantity_on_hand_total) <> SUM(available_to_promise_total)

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

```sql
SELECT 
sales_channel_enum_id AS SALES_CHANNEL,
COUNT(DISTINCT order_id) AS TOTAL_ORDERS,
SUM(grand_total) AS TOTAL_REVENUE,
entry_date AS REPORTING_PERIOD

FROM order_header
GROUP BY sales_channel_enum_id

```
