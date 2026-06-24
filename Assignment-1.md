# SQL Assignment 1

### 1 New Customers Acquired in June 2023

**Business Problem:**  
The marketing team ran a campaign in June 2023 and wants to see how many new customers signed up during that period.

**Fields to Retrieve:**  
- `PARTY_ID`  
- `FIRST_NAME`  
- `LAST_NAME`  
- `EMAIL`  
- `PHONE`  
- `ENTRY_DATE`

##QUERY 
```sql
SELECT 
PARTY_ID, FIRST_NAME, LAST_NAME, INFO_STRING, CONTACT_NUMBER
FROM PARTY p
LEFT JOIN PERSON USING(PARTY_ID)
LEFT JOIN PARTY_CONTACT_MECH USING(PARTY_ID)
LEFT JOIN CONTACT_MECH USING(CONTACT_MECH_ID)
LEFT JOIN POSTAL_ADDRESS USING(CONTACT_MECH_ID)
LEFT JOIN TELECOM_NUMBER USING(CONTACT_MECH_ID)
WHERE p.CREATED_STAMP BETWEEN '2023-06-01' AND '2023-06-30'
-- WHERE p.CREATED_STAMP >= '2023-06-01' AND p.CREATED_STAMP <= '2026-06-30';
```

### 2 List All Active Physical Products

**Business Problem:**  
Merchandising teams often need a list of all physical products to manage logistics, warehousing, and shipping.

**Fields to Retrieve:**  
- `PRODUCT_ID`  
- `PRODUCT_TYPE_ID`  
- `INTERNAL_NAME`

##Query 
```sql
  -- SELECT 
  -- PRODUCT_ID, PRODUCT_TYPE_ID, INTERNAL_NAME
  -- FROM PRODUCT
  -- JOIN PRODUCT_TYPE USING(PRODUCT_TYPE_ID)
  -- WHERE IS_PHYSICAL='Y' AND SALES_DISCONTINUATION_DATE IS NULL
  
  SELECT
    product_id,
    product_type_id,
    internal_name
FROM product
WHERE SALES_DISCONTINUATION_DATE IS NULL
  AND product_type_id NOT IN (
      SELECT product_type_id
      FROM product_type
      WHERE is_physical = "N"
  );
```

### 3 Products Missing NetSuite ID

**Business Problem:**  
A product cannot sync to NetSuite unless it has a valid NetSuite ID. The OMS needs a list of all products that still need to be created or updated in NetSuite.

**Fields to Retrieve:**  
- `PRODUCT_ID`  
- `INTERNAL_NAME`  
- `PRODUCT_TYPE_ID`  
- `NETSUITE_ID` (or similar field indicating the NetSuite ID; may be `NULL` or empty if missing)

##QUERY 
```sql
SELECT 
p.product_id AS PRODUCT_ID,
p.internal_name AS INTERNAL_NAME,
p.product_type_id AS PRODUCT_TYPE_ID,
gi.good_identification_type_id AS NETSUITE_ID

FROM product p
LEFT JOIN good_identification gi ON p.product_id = gi.product_id 
AND good_identification_type_id = 'NETSUITE_ID'
WHERE good_identification_type_id IS NULL
```

### 4 Product IDs Across Systems

**Business Problem:**  
To sync an order or product across multiple systems (e.g., Shopify, HotWax, ERP/NetSuite), the OMS needs to know each system’s unique identifier for that product. This query retrieves the Shopify ID, HotWax ID, and ERP ID (NetSuite ID) for all products.

**Fields to Retrieve:**  
- `PRODUCT_ID` (internal OMS ID)  
- `SHOPIFY_ID`  
- `HOTWAX_ID`  
- `ERP_ID` or `NETSUITE_ID` (depending on naming)

##QUERY
```sql
SELECT 
gi.product_id AS PRODUCT_ID,
CASE WHEN gi.good_identification_type_id = 'SHOPIFY_PROD_ID' THEN gi.id_value END AS SHOPIFY_ID,
gi.product_id AS HOTWAX_ID,
CASE WHEN gi.good_identification_type_id = 'NETSUITE_ID' THEN gi.id_value END AS ERP_ID

FROM good_identification gi
GROUP BY product_id
```

### 5 Completed Orders in August 2023

**Business Problem:**  
After running similar reports for a previous month, you now need all completed orders in August 2023 for analysis.

**Fields to Retrieve:**  
- `PRODUCT_ID`  
- `PRODUCT_TYPE_ID`  
- `PRODUCT_STORE_ID`  
- `TOTAL_QUANTITY`  
- `INTERNAL_NAME`  
- `FACILITY_ID`  
- `EXTERNAL_ID`  
- `FACILITY_TYPE_ID`  
- `ORDER_HISTORY_ID`  
- `ORDER_ID`  
- `ORDER_ITEM_SEQ_ID`  
- `SHIP_GROUP_SEQ_ID`

##QUERY
```sql
SELECT 
p.product_id AS PRODUCT_ID,
p.product_type_id AS PRODUCT_TYPE_ID,
oh.product_store_id AS PRODUCT_STORE_ID,
oi.quantity AS TOTAL_QUANTITY,
p.internal_name AS INTERNAL_NAME,
f.facility_id AS FACILITY_ID,
oh.external_id AS EXTERNAL_ID,
f.facility_type_id AS FACILITY_TYPE_ID,
ohis.order_history_id AS ORDER_HISTORY_ID,
oh.order_id AS ORDER_ID,
oi.order_item_seq_id AS ORDER_ITEM_SEQ_ID,
oisg.ship_group_seq_id AS SHIP_GROUP_SEQ_ID,

From order_header oh 
JOIN order_item oi USING(order_id)
JOIN product p USING(product_id)
JOIN order_item_ship_group oisg USING(order_id)
JOIN facility f ON oisg.facility_id = f.facility_id
JOIN order_history ohis USING(order_id)
JOIN order_status os USING(order_id)


WHERE os.status_datetime BETWEEN
      '2023-08-01T00:00:00.000000'
  AND '2023-08-31T23:59:59.999999'
```


### 7 Newly Created Sales Orders and Payment Methods

**Business Problem:**  
Finance teams need to see new orders and their payment methods for reconciliation and fraud checks.

**Fields to Retrieve:**  
- `ORDER_ID`
- `TOTAL_AMOUNT`
- `PAYMENT_METHOD`  
- `Shopify Order ID` (if applicable)

##QUERY
```sql
SELECT 
oh.order_id AS ORDER_ID,
oh.grand_total AS TOTAL_AMOUNT,
opp.payment_method_type_id AS PAYMENT_METHOD,
oh.external_id AS SHOPIFY_ORDER_ID

FROM order_header oh 
JOIN order_payment_preference opp USING(order_id)
WHERE oh.status_id = 'ORDER_CREATED'
```

### 8 Payment Captured but Not Shipped

**Business Problem:**  
Finance teams want to ensure revenue is recognized properly. If payment is captured but no shipment has occurred, it warrants further review.

**Fields to Retrieve:**  
- `ORDER_ID`  
- `ORDER_STATUS`  
- `PAYMENT_STATUS`  
- `SHIPMENT_STATUS`

##QUERY
```sql
SELECT order_id, oh.status_id AS order_status, opp.status_id as payment_status,ss.status_id as shipment_status 
FROM ORDER_HEADER oh
JOIN ORDER_PAYMENT_PREFERENCE opp USING(order_id)
JOIN ORDER_SHIPMENT os USING(order_id)
JOIN SHIPMENT_STATUS ss USING(shipment_id)

WHERE opp.status_id = 'PAYMENT_SETTLED'
AND ss.status_id <> 'SHIPMENT_SHIPPED'
AND oh.status_id <> 'ORDER_COMPLETED'
```
### 9 Orders Completed Hourly

**Business Problem:**  
Operations teams may want to see how orders complete across the day to schedule staffing.

**Fields to Retrieve:**  
- `TOTAL ORDERS`  
- `HOUR`

##QUERY
```sql
SELECT 
    COUNT(ORDER_ID) AS TOTAL_ORDERS,
    HOUR(ORDER_DATE) AS HOUR
FROM ORDER_HEADER
WHERE STATUS_ID = 'ORDER_COMPLETED'
GROUP BY HOUR(ORDER_DATE)
ORDER BY HOUR;
```

### 10 BOPIS Orders Revenue (Last Year)

**Business Problem:**  
**BOPIS** (Buy Online, Pickup In Store) is a key retail strategy. Finance wants to know the revenue from BOPIS orders for the previous year.

**Fields to Retrieve:**  
- `TOTAL ORDERS`  
- `TOTAL REVENUE`

##QUERY
```sql
SELECT COUNT(oh.order_id) AS TOTAL_ORDERS, CONCAT(SUM(COALESCE(oh.grand_total,0)-COALESCE(oa.amount)),'$') AS TOTAL_REVENUE
FROM ORDER_HEADER oh
JOIN ORDER_ITEM_SHIP_GROUP oisg USING(order_id) 
JOIN order_adjustment oa USING(order_id) 
WHERE oisg.shipment_method_type_id='STOREPICKUP'
AND oh.status_id <> 'ORDER_CANCEL'
```
### 11 Canceled Orders (Last Month)

**Business Problem:**  
The merchandising team needs to know how many orders were canceled in the previous month and their reasons.

**Fields to Retrieve:**  
- `TOTAL ORDERS`  
- `CANCELATION REASON`
##QUERY
```sql
SELECT COUNT(order_id), CHANGE_REASON 
FROM ORDER_STATUS
WHERE status_id = 'ORDER_CANCELLED' OR status_id = 'ITEM_CANCELLED'
AND  MONTH(last_updated_stamp) = MONTH(NOW() - INTERVAL 1 MONTH)
GRoUP BY (change_reason)
```
### 12 Product Threshold Value

**Business Problem**
The retailer has set a threshold value for products that are sold online, in order to avoid over selling. 

**Fields to Retrieve:**
- `PRODUCT ID`
- `THRESHOLD`

##QUERY
```sql
SELECT
product_id AS PRODUCT_ID,
minimum_stock AS THRESHOLD
FROM product_facility
```
