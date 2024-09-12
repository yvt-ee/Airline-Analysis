#### 1 Meal online review data

>Background: A new product was on sale, check the sale performance

````sql
SELECT t2.orderid,
       t2.userid,
       t1.remark,
       t2.created,
       t1.remark_level,
       t1.source, 
FROM   dw.fact_cms_ch_item_remark t1
       JOIN dw.fact_cms_remark t2
         ON t1.remarkid = t2.remarkid
WHERE t1.created >= To_date('2022-12-01', 'yyyy-mm-dd')
       AND t1.created < To_date('2023-01-01', 'yyyy-mm-dd'); 
````

#### 2 Cancellation policy exclude irregular flights

>Background: Check the cancellation fee charged for different reason

````sql
SELECT 
    DECODE(t1.log_code,
           5, 'Ticket Refund',
           6, 'Flight Change',
           20, 'Ticket Refund - Audit',
           22, 'Special Case Change - Audit',
           21, 'Name/ID Modification - Audit') AS Log_Type,
    TO_CHAR(t9.origin_std, 'yyyy-mm') AS Flight_Date,
    SUM(NVL(t1.fy, 0) * NVL(t2.r_com_rate, 1)) AS Actual_Cost,
    SUM(NVL(t1.ys_fy, 0) * NVL(t2.r_com_rate, 1)) AS Standard_Cost,
    COUNT(DISTINCT t1.flights_order_head_id) AS Unique_Order_Count
FROM 
    stg.s_cq_flights_head_history t1
JOIN 
    stg.s_cq_order_head t2 ON t1.flights_order_head_id = t2.flights_order_head_id
JOIN 
    stg.s_cq_flights t3 ON t2.flights_id = t3.flights_id
JOIN 
    stg.s_cq_user t5 ON t1.users_id = t5.users_id
                     AND t5.company_id = t3.company_id
JOIN 
    stg.s_cq_terminal t4 ON t5.terminal_id = t4.terminal_id
JOIN 
    stg.s_cq_order t6 ON t6.flights_order_id = t2.flights_order_id
JOIN 
    stg.s_cq_flights_segment_head t9 ON t2.segment_head_id = t9.segment_head_id
LEFT JOIN 
    dw.fact_foc_unnormal_flight t10 ON TRUNC(t9.origin_std) = t10.daylinedate
                                    AND t9.r_flights_no = t10.flightno
                                    AND t9.flights_segment = t10.originairport || t10.destairport
WHERE 
    t5.company_id = 0
    AND t10.dayflightid IS NULL
    AND t9.origin_std >= TO_DATE('2019-11-01', 'yyyy-mm-dd')
    AND t9.origin_std < TO_DATE('2022-01-01', 'yyyy-mm-dd')
    AND t1.log_code IN (20, 22)
GROUP BY 
    DECODE(t1.log_code,
           5, 'Ticket Refund',
           6, 'Flight Change',
           20, 'Ticket Refund - Audit',
           22, 'Special Case Change - Audit',
           21, 'Name/ID Modification - Audit'),
    TO_CHAR(t9.origin_std, 'yyyy-mm');
````

#### 3 Customer Information

>Background: Customer Profile change

````sql

 select to_char(t1.flights_date,'yyyy'),
       case when t1.seats_name in('B','G','G1','G2','O') then 'BGO'
       else 'nonBGO' end cabin type,
       case
         when t1.ahead_days <= 3 then
          '00-03'
         when t1.ahead_days <= 7 then
          '04-07'
         when t1.ahead_days <= 15 then
          '08-15'
         when t1.ahead_days <= 30 then
          '15-30'
         when t1.ahead_days <= 60 then
          '31-60'
         when t1.ahead_days <= 90 then
          '61-90'
         when t1.ahead_days > 90 then
          '90+'
       end purchaseleadtime,
       t1.nationflag routetype,
       case when t5.nationality like '%inland%' then 'inland'
       else 'outland' end nationality,
        case when t5.nationality like '%inland%' then t5.cust_province
       else '-' end province,
       case when t5.nationality like '%inland%' then t5.cust_city
       else '-' end city,
       t1.gender sex, 
       case when getage(t1.flights_date,t1.birthday)<12 then '<12'
            when getage(t1.flights_date,t1.birthday)<18 then '12~17'
            when getage(t1.flights_date,t1.birthday)<24 then '18~23'
            when getage(t1.flights_date,t1.birthday)<30 then '24~29'
            when getage(t1.flights_date,t1.birthday)<40 then '30~39'
            when getage(t1.flights_date,t1.birthday)<50 then '40~49'
            when getage(t1.flights_date,t1.birthday)<60 then '50~59'
            when getage(t1.flights_date,t1.birthday)<70 then '60~69'
            when getage(t1.flights_date,t1.birthday)>=70 then '70+' 
            else '-' end age,     
       count(1) sales,
       sum(t1.ticket_price) saleamount,
       sum(t1.price) announcedprice,
       sum(t1.insurce_fee+t1.other_fee) supplementaryincome
  from dw.fact_order_detail t1
  join dw.da_flight t2 on t1.segment_head_id = t2.segment_head_id
  left join dw.bi_order_region t5 on t1.flights_order_head_id=t5.flights_order_head_id
 where t1.flights_date >= to_date('2017-01-01', 'yyyy-mm-dd')
   and t1.flights_date < to_date('2020-10-01', 'yyyy-mm-dd')
   and t1.whole_flight like '9C%'
   and t1.seats_name is not null
 group by to_char(t1.flights_date,'yyyy'),
       case when t1.seats_name in('B','G','G1','G2','O') then 'BGO'
       else '非BGO' end ,
       case
         when t1.ahead_days <= 3 then
          '00-03'
         when t1.ahead_days <= 7 then
          '04-07'
         when t1.ahead_days <= 15 then
          '08-15'
         when t1.ahead_days <= 30 then
          '15-30'
         when t1.ahead_days <= 60 then
          '31-60'
         when t1.ahead_days <= 90 then
          '61-90'
         when t1.ahead_days > 90 then
          '90+'
       end ,
       t1.nationflag ,
       case when t5.nationality like '%inland%' then 'inland'
       else 'outland' end ,
        case when t5.nationality like '%inland%' then t5.cust_province
       else '-' end ,
       case when t5.nationality like '%inland%' then t5.cust_city
       else '-' end ,
       t1.gender , 
       case when getage(t1.flights_date,t1.birthday)<12 then '<12'
            when getage(t1.flights_date,t1.birthday)<18 then '12~17'
            when getage(t1.flights_date,t1.birthday)<24 then '18~23'
            when getage(t1.flights_date,t1.birthday)<30 then '24~29'
            when getage(t1.flights_date,t1.birthday)<40 then '30~39'
            when getage(t1.flights_date,t1.birthday)<50 then '40~49'
            when getage(t1.flights_date,t1.birthday)<60 then '50~59'
            when getage(t1.flights_date,t1.birthday)<70 then '60~69'
            when getage(t1.flights_date,t1.birthday)>=70 then '70+' 
            else '-' end;
````



#### 4 Customer behavior - Flight booking

>Background: For those who have taken two or more flights, & whether the time period of the first and second flights is the same


````sql

 
select /*+parallel(4) */
 to_char(tb1.flights_date,'yyyymm'),tb1.flights_city_name,tb1.origin_std,tb1.dest_sta,
tb1.origin_std2,tb1.dest_sta2,
case when tb1.origin_std=tb1.origin_std2
and  tb1.dest_sta=tb1.dest_sta2 then 'same'
when tb1.origin_std=tb1.origin_std2 then 'startsame'
when  tb1.dest_sta=tb1.dest_sta2 then 'arrivesame'
else 'different' end,
tb1.flightdate2-tb1.flights_date intervaltime,
count(1)
from(
select h1.traveller_name,h1.codeno,h1.flights_date,h1.flights_city_name,h1.origin_std,
h1.dest_sta,h2.flights_date flightdate2,h2.origin_std origin_std2,h2.dest_sta dest_sta2,
row_number()over(partition by h1.traveller_name,h1.codeno,h1.flights_city_name  order by h1.flights_date,h2.flights_date) rid
from(
select t1.traveller_name,t1.codetype,t1.codeno,t1.flights_date,f.flights_city_name,
       case  
                when to_char(f.origin_std, 'hh24') >= '00' and
                              to_char(f.origin_std, 'hh24') <= '03' then
                          'redeye'
                         when to_char(f.origin_std, 'hh24') >= '04' and
                              to_char(f.origin_std, 'hh24') <= '07' then
                          'morning'
                         when to_char(f.origin_std, 'hh24') >= '08' and
                              to_char(f.origin_std, 'hh24') <= '11' then
                          'forenoon'
                         when to_char(f.origin_std, 'hh24') >= '12' and
                              to_char(f.origin_std, 'hh24') <= '15' then
                          'afternoon'
                         when to_char(f.origin_std, 'hh24') >= '16' and
                              to_char(f.origin_std, 'hh24') <= '19' then
                          'night'
                         else
                          'latenight'
                       end origin_std,
                       case                        
                         when to_char(f.dest_sta, 'hh24') >= '00' and
                              to_char(f.dest_sta, 'hh24') <= '03' then
                          'redeye'
                         when to_char(f.dest_sta, 'hh24') >= '04' and
                              to_char(f.dest_sta, 'hh24') <= '07' then
                          'morning'
                         when to_char(f.dest_sta, 'hh24') >= '08' and
                              to_char(f.dest_sta, 'hh24') <= '11' then
                          'forenoon'
                         when to_char(f.dest_sta, 'hh24') >= '12' and
                              to_char(f.dest_sta, 'hh24') <= '15' then
                          'afternoon'
                         when to_char(f.dest_sta, 'hh24') >= '16' and
                              to_char(f.dest_sta, 'hh24') <= '19' then
                          'night'
                         else
                          'latenight'
                       end dest_sta
      from dw.fact_order_detail t1
      join dw.da_flight f on t1.segment_head_id=f.segment_head_id
      where f.flights_city_name like '%Shanghai%'
      and regexp_like(f.flights_segment_name,'(Shenzheng)|(Guangzhou)')
      and t1.flights_date>=to_date('2019-03-31','yyyy-mm-dd')
      and t1.flights_date< to_date('2020-01-01','yyyy-mm-dd')
      and t1.flag_Id=40
      and t1.seats_name not in('B','G','G1','G2','O'))h1
      
      join 
      (select t1.traveller_name,t1.codetype,t1.codeno,t1.flights_date,f.flights_city_name,
       case  
                when to_char(f.origin_std, 'hh24') >= '00' and
                              to_char(f.origin_std, 'hh24') <= '03' then
                          'redeye'
                         when to_char(f.origin_std, 'hh24') >= '04' and
                              to_char(f.origin_std, 'hh24') <= '07' then
                          'morning'
                         when to_char(f.origin_std, 'hh24') >= '08' and
                              to_char(f.origin_std, 'hh24') <= '11' then
                          'forenoon'
                         when to_char(f.origin_std, 'hh24') >= '12' and
                              to_char(f.origin_std, 'hh24') <= '15' then
                          'afternoon'
                         when to_char(f.origin_std, 'hh24') >= '16' and
                              to_char(f.origin_std, 'hh24') <= '19' then
                          'night'
                         else
                          'latenight'
                       end origin_std,
                       case                        
                         when to_char(f.dest_sta, 'hh24') >= '00' and
                              to_char(f.dest_sta, 'hh24') <= '03' then
                          'redeye'
                         when to_char(f.dest_sta, 'hh24') >= '04' and
                              to_char(f.dest_sta, 'hh24') <= '07' then
                          'morning'
                         when to_char(f.dest_sta, 'hh24') >= '08' and
                              to_char(f.dest_sta, 'hh24') <= '11' then
                          'forenoon'
                         when to_char(f.dest_sta, 'hh24') >= '12' and
                              to_char(f.dest_sta, 'hh24') <= '15' then
                          'afternoon'
                         when to_char(f.dest_sta, 'hh24') >= '16' and
                              to_char(f.dest_sta, 'hh24') <= '19' then
                          'night'
                         else
                          'latenight'
                       end dest_sta
      from dw.fact_order_detail t1
      join dw.da_flight f on t1.segment_head_id=f.segment_head_id
      where f.flights_city_name like '%Shanghai%'
      and regexp_like(f.flights_segment_name,'(Shenzheng)|(Guangzhou)')
      and t1.flights_date>=to_date('2019-03-31','yyyy-mm-dd')
      and t1.flights_date< to_date('2020-01-01','yyyy-mm-dd')
      and t1.flag_Id=40
      and t1.seats_name not in('B','G','G1','G2','O'))h2 on h1.traveller_name=h2.traveller_name and h1.codeno=h2.codeno 
      and h1.flights_city_name=h2.flights_city_name and h1.flights_date< h2.flights_date)tb1
      where tb1.rid=1
      group by to_char(tb1.flights_date,'yyyymm'),tb1.flights_city_name,tb1.origin_std,tb1.dest_sta,
tb1.origin_std2,tb1.dest_sta2,
case when tb1.origin_std=tb1.origin_std2
and  tb1.dest_sta=tb1.dest_sta2 then 'same'
when tb1.origin_std=tb1.origin_std2 then 'startsame'
when  tb1.dest_sta=tb1.dest_sta2 then 'arrivesame'
else '不同' end,tb1.flightdate2-tb1.flights_date;
````


#### 5 Cancellation policy exclude irregular flights

>Background:
>1. What are the popular international routes and destinations during the Spring Festival this year? Are passenger traffic to these countries up or down compared to last year, and how has the year-on-year growth rate changed? 
>2. Which international routes were hot last Spring Festival, but cooled down this year? Is there anything in the data? * / 

````sql
select year,flights_segment_name,nationflag,sum(oversale),sum(ticketnum)
from (
select  '2020' year,
t2.flights_segment_name,t1.segment_head_id,t2.oversale,t1.nationflag,
count(1) ticketnum
from dw.fact_order_detail t1
join dw.da_flight t2 on t1.segment_head_id=t2.segment_head_id
where t1.flights_date>=to_date('2020-01-10','yyyy-mm-dd')
 and t1.flights_date< to_date('2020-02-19','yyyy-mm-dd')
 and t1.company_id=0
 and t2.flag<>2
 and t1.seats_name is not null
 and t2.dest_country_id>0
 group by  t2.flights_segment_name,t1.segment_head_id,t2.oversale,t1.nationflag
 
 union all
 
select  '2019' year, t2.flights_segment_name,t1.segment_head_id,t2.oversale,t1.nationflag,count(1)
from dw.fact_order_detail t1
join dw.da_flight t2 on t1.segment_head_id=t2.segment_head_id
where t1.flights_date>=to_date('2019-01-21','yyyy-mm-dd')
 and t1.flights_date<= to_date('2019-03-01','yyyy-mm-dd')
 and t1.company_id=0
 and t2.flag<>2
 and t1.order_day< to_date('2019-01-27','yyyy-mm-dd')
 and t1.seats_name is not null
 and t2.dest_country_id>0
 group by  t2.flights_segment_name,t1.segment_head_id,t2.oversale,t1.nationflag)h1
 group by year,flights_segment_name,nationflag;
````


