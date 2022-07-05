# johns-repository
----- Metros Table ------

select id ---- metro id
, name ---- metro name
, status ---- status either active or inactive. usually only look at active metros
, region ---- metro region
from og_views.metros
where status = 'active'
order by 1

;

------ Stores Table ------

select id ---- store id
, name ---- store name
, active ---- looks to see if store is active or inactive
, sells_alcohol ---- displays if store is allowed to sell alcohol
from og_views.stores
where active = true
order by 1

;

------ Categories Table ------

select *
from og_views.categories
limit 100

;

------ Crpytex Category Hierarchy Table ------

with recursive category_list as
    (select id as category_id, iff
         (parent_id = 0,
            null,
            parent_id) parent_id,
        name as category_name,
        category_type
    from ng_views.shipt_category_service_categories
    where category_type = 'Cryptex' --or 'Retailer Specific'
    and STATUS = 'active'
    and DELETED_AT is NULL),
q as (
    select category_id, 1 as taxonomy_level, parent_id, category_name
    from category_list
    where parent_id is null
    union all
    select c.category_id, q.taxonomy_level + 1, c.parent_id, c.category_name
    from q
    join category_list as c on c.parent_id = q.category_id)
select distinct q.category_id as T1_id, q.category_name as T1_name,
                q1.category_id as T2_id, q1.category_name as T2_name,
                q2.category_id as T3_id, q2.category_name as T3_name,
                q3.category_id as T4_id, q3.category_name as T4_name,
                q4.category_id as T5_id, q4.category_name as T5_name from q
left join q q1 on q.category_id = q1.parent_id
left join q q2 on q1.category_id = q2.parent_id
left join q q3 on q2.category_id = q3.parent_id
left join q q4 on q3.category_id = q4.parent_id
where q.taxonomy_level = 1
order by 2,4,6,8,10 asc nulls last

;

------ Order Lines Table ------


select id ---- order lines id
, order_id ---- order id
, actual_qty ----- number of items purcahsed
, actual_product_id ----- product purchased
, actual_product_type ---- displays product or custom product
, saved_product_on_sale ---- displays if product was on sale
, saved_product_price ---- marked up amount on product
, saved_product_cost ---- in store cost
, saved_product_price/saved_product_cost-1 as effective_markup ---- markup calculation. won't be an exact match to the markup set due to rounding everything to the nearest 9 cents
from og_views.order_lines
where order_id = '65049599'
order by 1

;
------ Order Financials Table ------

select
  o.id
  ,created_at_central
  ,delivered_at_central
  ,count(*)
  ,avg(ue) as ue
  ,avg(gmv) as gmv
  ,avg(grocery_revenue) as grocery_revenue 
  ,avg(pex_rebate) as pex_rebate
  ,avg(delivery_fee) as delivery_fee
  ,avg(cost) as cost
  ,avg(shopper_pay) as shopper_pay
  ,avg(promo_pay) as promo_pay
  ,avg(incentive_pay) as incentive_pay
  ,avg(ch_amount) as credits
  ,avg(r_amount) as refunds
  ,avg(transaction_fees) as transaction_fees
  ,avg(fraud) as fraud
  ,avg(hosting) as hosting
  ,avg(x_team) as x_team
  ,avg(insurance) as insurance
  ,avg(cpg_revenue) as CPG
  ,avg(membership_revenue)
from order_financials o
join og_views.metros m on o.metro_id = m.id 
join og_views.stores s on s.id = o.store_id
Where order_type in ('Marketplace','White Label','Platform') -- Marketplace, White Label, Platform, Envoy
And membership_type in ('Membership','Single PPO','3-pack','5-pack')
And store_id in (1) -- Publix
And delivered_at_central::date between '2020-01-01' and '2020-12-31'
Group by 1;




     date_trunc('Month', convert_timezone('US/Central', ord.delivered_at))::date as Month
     --, P.DISPLAY_NAME AS Display_Name
     --, s.name
     --, p.UPC as UPC
     --, p.product_id as ID
     , c.name as Category
     , c.id as category_id
     --, case
     --   when cc.cpg_name = 'Private Label' then 'Private Label'
     --   else 'Other'
     --   end 
     --       as PL_OTHER
     --, s.name as Retailer
     --, s.store_type as Channel
     --, count(distinct sl.id) as locations
     --, p.brand_name as Brand
     --, s.name as Retailer
     --, c.parent_id as Tier_1
     , round(SUM((o.saved_product_price) * (o.actual_qty)),2) AS GMV
     , SUM(o.actual_qty) AS Qty
     , count(distinct(o.order_id)) as Orders
---------------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------------
--SECTION TWO: WHERE YOU GET THE STUFF
     /* JOINS FOR NG SCHEMA */
FROM og_views.order_lines o                                                                     --order_lines o = og table with ORDER details FOR EACH ITEM within an order                            
    JOIN og_views.orders ord                                                                    --orders ord = og table showing ORDER details FOR THE ORDER ITSELF
        ON o.order_id = ord.id
            AND ord.status = 'delivered' --FILTERING OUT UNDELIVERED/CANCELED ORDERS
            AND ord.is_external_platform_order = false --FILTERING OUT PLATFORM
            AND ord.partner_id is null --FILTERING OUT WHITE LABEL
            AND o.actual_product_type = 'Product' --FILTERS TO PRODUCTS ONLY, NO P
            
    JOIN ng_views.shipt_product_service_products p                                              --shipt_product_service_products p = catalog products and general information                              
        ON o.actual_product_id = p.product_id
            AND p.deleted_at is null
    
    --JOIN og_views.metros m                                                                    --metros m = Shipt designated metropolitan areas; they have names and codes
    --    ON ord.metro_id = m.id
        
    JOIN data_science.prod_codes pc                                                             --prod_codes pc = table where QA is done and CPG_ID is added to the product                 
        ON p.product_id = pc.product_id
        AND qa_complete = 'True'                  
        
    JOIN data_science.cpg_codes cc                                                              --cpg_codes cc = table where CPG information is attached                 
        ON pc.cpg_id = cc.cpg_id
        
    JOIN og_views.store_locations sl                                                            --store_locations sl = table containing store LOCATION information - individual stores, e.g. Target #1874
        ON sl.id = ord.store_location_id
    --JOIN og_views.stores s                                                                    --stores s = retailer table, e.g. Target, Meijer, Publix, etc.
    --    ON s.id = ord.store_id
        
    JOIN ng_views.shipt_category_service_product_categorizations scsp                           --shipt_category_service_product_categorizations scsp = table where category_ids are assigned            
        ON p.product_id = scsp.product_id
        
    JOIN ng_views.shipt_category_service_categories c                                           --shipt_category_service_categories c = table where granular category information is added
      ON c.id = scsp.category_id
          AND c.category_type = 'Cryptex'
          
     /* END JOINS FOR NG SCHEMA */
 
---------------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------------
--SECTION 3: THE CONDITIONS/PARAMETERS FOR THE STUFF
WHERE 
      CAST(ord.delivered_at AS DATE) >= '2022-01-01'
      --AND d2.cpg_name = 'Bob''s Red Mill'
      --AND s.store_type = 'DrugConvenience'
      --AND o.actual_product_type = 'Product'
      AND sl.state in ('AL','FL')
      AND category_id in (4958, 4961, 4912, 4913, 4255, 5895)
      --AND c.name = 'Food'
      --AND p.brand_name like any ('Enfa%','Apta%','Simil%')
      
---------------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------------
--SECTION 4: HOW YOU WANT TO SEE YOUR STUFF
GROUP BY 1,2,3 --always group by everything that is not a calculated value
ORDER BY 1,4 desc--,3,4,5,7 desc 
