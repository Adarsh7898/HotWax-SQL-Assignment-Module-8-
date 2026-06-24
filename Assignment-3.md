# SQL Assignment 3

### 1 Completed Sales Orders (Physical Items)

**Business Problem:**  
Merchants need to track only physical items (requiring shipping and fulfillment) for logistics and shipping-cost analysis.

**Fields to Retrieve:**  
- `ORDER_ID`  
- `ORDER_ITEM_SEQ_ID`  
- `PRODUCT_ID`  
- `PRODUCT_TYPE_ID`  
- `SALES_CHANNEL_ENUM_ID`  
- `ORDER_DATE`  
- `ENTRY_DATE`  
- `STATUS_ID`  
- `STATUS_DATETIME`  
- `ORDER_TYPE_ID`  
- `PRODUCT_STORE_ID`  

##QUERY
```sql
SELECT DISTINCT 
oh.order_id AS ORDER_ID,
oi.order_item_seq_id AS ORDER_ITEM_SEQ_ID,
oi.product_id AS PRODUCT_ID,
p.product_id AS PRODUCT_TYPE_ID,
oh.sales_channel_enum_id AS SALES_CHANNEL_ENUM_ID,
oh.order_date AS ORDER_DATE,
oh.entry_date AS ENTRY_DATE,
oh.status_id AS STATUS_ID,
os.status_datetime AS STATUS_DATETIME,
oh.order_type_id AS ORDER_TYPE_ID,
oh.product_store_id AS PRODUCT_STORE_ID

FROM order_header oh
JOIN order_item oi USING(order_id)
JOIN order_status os USING(order_id)
JOIN product p USING(product_id)

WHERE oh.order_type_id = 'SALES_ORDER'
AND oh.status_id = 'ORDER_COMPLETED'
AND p.product_type_id = 'FINISHED_GOOD' 
AND oi.status_id <> "item_cancelled"
```




### 2 Completed Return Items

**Business Problem:**  
Customer service and finance often need insights into **returned items** to manage refunds, replacements, and inventory restocking.

**Fields to Retrieve:**  
- `RETURN_ID`  
- `ORDER_ID`  
- `PRODUCT_STORE_ID`  
- `STATUS_DATETIME`  
- `ORDER_NAME`  
- `FROM_PARTY_ID`
- `RETURN_DATE`  
- `ENTRY_DATE`  
- `RETURN_CHANNEL_ENUM_ID`

##QUERY
```sql
SELECT
ri.return_id AS RETURN_ID,
ri.order_id AS ORDER_ID,
oh.product_store_id AS PRODUCT_STORE_ID,
rh.status_ID AS RETURN_STATUS,
rs.status_datetime AS STATUS_DATETIME,
oh.order_name AS ORDER_NAME,
rh.from_party_id AS FROM_PARTY_ID,
rh.return_date AS RETURN_DATE,
rh.entry_date AS ENTRY_DATE,
rh.return_channel_enum_id AS RETURN_CHANNEL_ENUM_ID

FROM return_header rh 
JOIN return_item ri USING(return_id)
JOIN return_status rs USING(return_id)
JOIN order_header oh ON ri.order_id = oh.order_id
WHERE rh.status_id = 'RETURN_COMPLETED'
```



### 3 Single-Return Orders (Last Month)

**Business Problem:**  
The mechandising team needs a list of orders that only have one return.

**Fields to Retrieve:**  
- `PARTY_ID`  
- `FIRST_NAME`
##QUERY
```sql
SELECT DISTINCT
per.party_id AS PARTY_ID,
per.first_name AS FIRST_NAME,
role.order_id 

FROM return_item ri 
JOIN order_role role USING(order_id)
JOIN person per ON role.party_id = per.party_id
WHERE return_id IN (
    SELECT return_id FROM return_item 
    GROUP BY order_id
    HAVING COUNT(order_id)=1

)
```


### 4 Returns and Appeasements 

**Business Problem:**  
The retailer needs the total amount of items, were returned as well as how many appeasements were issued.

**Fields to Retrieve:**  
- `TOTAL RETURNS`
- `RETURN $ TOTAL`
- `TOTAL APPEASEMENTS`
- `APPEASEMENTS $ TOTAL`

##QUERY
```sql
SELECT COUNT(RI.return_id) AS total_return,
    SUM(
        CASE
            WHEN ra.return_adjustment_type_id = 'Appeasement' THEN 1  ELSE 0 END
    ) AS appeasement_count
FROM RETURN_ITEM RI
JOIN RETURN_ADJUSTMENT ra ON RI.return_id = RA.return_id
GROUP BY ra.return_adjustment_id;
```

### 5 Detailed Return Information

**Business Problem:**  
Certain teams need granular return data (reason, date, refund amount) for analyzing return rates, identifying recurring issues, or updating policies.

**Fields to Retrieve:**  
- `RETURN_ID`  
- `ENTRY_DATE`  
- `RETURN_ADJUSTMENT_TYPE_ID` (refund type, store credit, etc.)  
- `AMOUNT`  
- `COMMENTS`  
- `ORDER_ID`  
- `ORDER_DATE`  
- `RETURN_DATE`  
- `PRODUCT_STORE_ID`

##QUERY 
```sql
SELECT 
rh.return_id AS RETURN_ID,
rh.entry_date AS ENTRY_DATE,
ra. return_adjustment_type_id AS RETURN_ADJUSTMENT_TYPE_ID,
ra.amount AS AMOUNT,
ra.comments COMMENTS,
oh.order_id AS ORDER_ID,
oh.order_date AS ORDER_DATE,
rh.return_date AS RETURN_DATE,
oh.product_Store_id PRODUCT_STORE_ID
FROM return_header rh 
JOIN return_item ri USING(return_id)
LEFT JOIN order_header oh ON ri.order_id=oh.order_id
LEFT JOIN return_adjustment ra ON ra.return_id = rh.return_id;
```



### 6 Orders with Multiple Returns

**Business Problem:**  
Analyzing orders with multiple returns can identify potential fraud, chronic issues with certain items, or inconsistent shipping processes.

**Fields to Retrieve:**  
- `ORDER_ID`  
- `RETURN_ID`  
- `RETURN_DATE`  
- `RETURN_REASON`  
- `RETURN_QUANTITY`

##QUERY
```sql
SELECT 
ri.order_id AS ORDER_ID,
rh.return_id AS RETURN_ID,
rh.return_date AS RETURN_DATE,
ri.return_reason_id AS RETURN_REASON,
ri.return_quantity AS RETURN_QUANTITY

FROM return_header rh 
JOIN return_item ri USING(return_id)
WHERE rh.return_id IN(
    SELECT return_id
    FROM return_item
    GROUP BY return_id
    HAVING COUNT(order_id)>1

)
```

### 7 Store with Most One-Day Shipped Orders (Last Month)

**Business Problem:**  
Identify which facility (store) handled the highest volume of “one-day shipping” orders in the previous month, useful for operational benchmarking.

**Fields to Retrieve:**  
- `FACILITY_ID`
- `FACILITY_NAME`  
- `TOTAL_ONE_DAY_SHIP_ORDERS`  
- `REPORTING_PERIOD`

##QUERY
```sql
SELECT
f.facility_id AS FACILITY_ID,
f.facility_name AS FACILITY_NAME,
COUNT(DISTINCT oh.order_id) AS TOTAL_ONE_DAY_SHIP_ORDERS,
oh.order_date AS REPORTING_PERIOD
FROM order_header oh 
JOIN order_item_ship_group oisg USING(order_id)
JOIN facility f ON oisg.facility_id = f.facility_id
WHERE oisg.shipment_method_type_id = 'NEXT_DAY'
AND oh.ORDER_DATE >= DATE_SUB(CURRENT_DATE, INTERVAL 1 MONTH)
GROUP BY FACILITY_ID
ORDER BY TOTAL_ONE_DAY_SHIP_ORDERS DESC
LIMIT 1
```

### 8 List of Warehouse Pickers

**Business Problem:**  
Warehouse managers need a list of employees responsible for picking and packing orders to manage shifts, productivity, and training needs.

**Fields to Retrieve:**  
- `PARTY_ID` (or Employee ID)  
- `NAME` (First/Last)  
- `ROLE_TYPE_ID` (e.g., “WAREHOUSE_PICKER”)  
- `FACILITY_ID` (assigned warehouse)  
- `STATUS` (active or inactive employee)

##QUERY
```sql
SELECT 
fp.party_id AS PARTY_ID, 
CONCAT(p.first_name,' ',p.last_name) AS NAME,
fp.role_type_id AS ROLE_TYPE_ID,
fp.facility_id AS FACILITY_ID,
pt.status_id AS STATUS 

FROM facility_party fp 
JOIN party pt USING(party_id)
JOIN person p USING(party_id)

WHERE fp.role_type_id = 'WAREHOUSE_PICKER'
```
---

### 9 Total Facilities That Sell the Product

**Business Problem:**  
Retailers want to see how many (and which) facilities (stores, warehouses, virtual sites) currently offer a product for sale.

**Fields to Retrieve:**  
- `PRODUCT_ID`  
- `PRODUCT_NAME` (or `INTERNAL_NAME`)  
- `FACILITY_COUNT` (number of facilities selling the product)  
- (Optionally) a **list of FACILITY_IDs** if more detail is needed

##QUERY
```sql
SELECT 
    p.product_id AS PRODUCT_ID,
    p.internal_name AS PRODUCT_NAME,
    COUNT(pf.facility_id) AS FACILITY_COUNT,
    GROUP_CONCAT(pf.facility_id) AS LIST_OF_FACILITIES
FROM product_facility pf
JOIN product p USING(product_id)
GROUP BY 
    p.product_id,
    p.internal_name;
```

---

### 10 Total Items in Various Virtual Facilities

**Business Problem:**  
Retailers need to study the relation of inventory levels of products to the type of facility it's stored at. Retrieve all inventory levels for products at locations and include the facility type Id. Do not retrieve facilities that are of type Virtual.

**Fields to Retrieve:**  
- `PRODUCT_ID`  
- `FACILITY_ID`
- `FACILITY_TYPE_ID`
- `QOH` (Quantity on Hand)  
- `ATP` (Available to Promise)

##QUERY
```sql
SELECT ii.product_id AS PRODUCT_ID,
ii.facility_id AS FACILITY_ID,
f.facility_type_id AS FACILITY_TYPE_ID,
ii.Quantity_on_hand_total AS Quantity_ON_Hand,
ii.Available_to_promise_total AS Available_To_Promise

FROM inventory_item ii
JOIN facility f USING(facility_id)

WHERE f.FACILITY_TYPE_ID NOT IN (
    SELECT FACILITY_TYPE_ID
    FROM FACILITY_TYPE
    WHERE PARENT_TYPE_ID = 'VIRTUAL_FACILITY'
);
```

### 11 Transfer Orders Without Inventory Reservation

**Business Problem:**  
When transferring stock between facilities, the system should reserve inventory. If it isn’t reserved, the transfer may fail or oversell.

**Fields to Retrieve:**  
- `TRANSFER_ORDER_ID`  
- `FROM_FACILITY_ID`  
- `TO_FACILITY_ID`  
- `PRODUCT_ID`  
- `REQUESTED_QUANTITY`  
- `RESERVED_QUANTITY`  
- `TRANSFER_DATE`  
- `STATUS`


##QUERY
```sql
SELECT oi.order_id AS TRANSFER_ORDER_ID, 
s.origin_facility_id AS FROM_FACILITY_ID,
s.destination_facility_id AS TO_FACILITY_ID,
oi.product_id AS PRODUCT_ID,
oi.quantity AS REQUESTED_QUANTITY,
oisgir.quantity AS RESERVED_QUANTITY,
sr.datetime_received AS TRANSFER_DATE,
oh.status_id AS STATUS

FROM order_header oh
JOIN order_item oi USING(order_id)
LEFT JOIN order_item_ship_grp_inv_res oisgir USING(order_id)
JOIN shipment s ON oh.order_id=s.primary_order_id
JOIN shipment_receipt sr USING(order_id)

WHERE oh.order_type_id='TRANSFER_ORDER'
AND oh.order_id NOT IN (
SELECT order_id FROM order_item_ship_grp_inv_res
)
```

### 12 Orders Without Picklist

**Business Problem:**  
A picklist is necessary for warehouse staff to gather items. Orders missing a picklist might be delayed and need attention.

**Fields to Retrieve:**  
- `ORDER_ID`  
- `ORDER_DATE`  
- `ORDER_STATUS`  
- `FACILITY_ID`
- `DURATION` (How long has the order been assigned at the facility)

##QUERY
```sql

### 11 Transfer Orders Without Inventory Reservation

**Business Problem:**  
When transferring stock between facilities, the system should reserve inventory. If it isn’t reserved, the transfer may fail or oversell.

**Fields to Retrieve:**  
- `TRANSFER_ORDER_ID`  
- `FROM_FACILITY_ID`  
- `TO_FACILITY_ID`  
- `PRODUCT_ID`  
- `REQUESTED_QUANTITY`  
- `RESERVED_QUANTITY`  
- `TRANSFER_DATE`  
- `STATUS`


##QUERY
```sql
SELECT oi.order_id AS TRANSFER_ORDER_ID, 
s.origin_facility_id AS FROM_FACILITY_ID,
s.destination_facility_id AS TO_FACILITY_ID,
oi.product_id AS PRODUCT_ID,
oi.quantity AS REQUESTED_QUANTITY,
oisgir.quantity AS RESERVED_QUANTITY,
sr.datetime_received AS TRANSFER_DATE,
oh.status_id AS STATUS

FROM order_header oh
JOIN order_item oi USING(order_id)
LEFT JOIN order_item_ship_grp_inv_res oisgir USING(order_id)
JOIN shipment s ON oh.order_id=s.primary_order_id
JOIN shipment_receipt sr USING(order_id)

WHERE oh.order_type_id='TRANSFER_ORDER'
AND oh.order_id NOT IN (
SELECT order_id FROM order_item_ship_grp_inv_res
)
```

### 12 Orders Without Picklist

**Business Problem:**  
A picklist is necessary for warehouse staff to gather items. Orders missing a picklist might be delayed and need attention.

**Fields to Retrieve:**  
- `ORDER_ID`  
- `ORDER_DATE`  
- `ORDER_STATUS`  
- `FACILITY_ID`
- `DURATION` (How long has the order been assigned at the facility)

##QUERY
```sql
SELECT 
oh.order_id AS ORDER_ID,
oh.order_date AS ORDER_DATE,
oh.status_id AS ORDER_STATUS,
oisg.facility_id,
DATEDIFF(CURRENT_TIMESTAMP, os. STATUS_DATETIME) AS DURATION_ASSIGNED_DAYS
FROM order_header oh
JOIN order_item_ship_group oisg USING(order_id)
LEFT JOIN order_status os ON oh.order_id = os.order_id AND os.status_id='ORDER_APPROVED'
WHERE oh.status_id = 'ORDER_APPROVED'
AND NOT EXISTS(
SELECT 1
FROM picklist_item pi
WHERE pi.order_id = oh.order_id)
```
