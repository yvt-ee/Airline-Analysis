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
    AND t9.origin_std >= TO_DATE('2021-11-01', 'yyyy-mm-dd')
    AND t9.origin_std < TO_DATE('2023-01-01', 'yyyy-mm-dd')
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
 where t1.flights_date >= to_date('2019-01-01', 'yyyy-mm-dd')
   and t1.flights_date < to_date('2022-10-01', 'yyyy-mm-dd')
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
      and t1.flights_date>=to_date('2021-03-31','yyyy-mm-dd')
      and t1.flights_date< to_date('2022-01-01','yyyy-mm-dd')
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
      and t1.flights_date>=to_date('2021-03-31','yyyy-mm-dd')
      and t1.flights_date< to_date('2022-01-01','yyyy-mm-dd')
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
select  '2022' year,
t2.flights_segment_name,t1.segment_head_id,t2.oversale,t1.nationflag,
count(1) ticketnum
from dw.fact_order_detail t1
join dw.da_flight t2 on t1.segment_head_id=t2.segment_head_id
where t1.flights_date>=to_date('2022-01-10','yyyy-mm-dd')
 and t1.flights_date< to_date('2022-02-19','yyyy-mm-dd')
 and t1.company_id=0
 and t2.flag<>2
 and t1.seats_name is not null
 and t2.dest_country_id>0
 group by  t2.flights_segment_name,t1.segment_head_id,t2.oversale,t1.nationflag
 
 union all
 
select  '2021' year, t2.flights_segment_name,t1.segment_head_id,t2.oversale,t1.nationflag,count(1)
from dw.fact_order_detail t1
join dw.da_flight t2 on t1.segment_head_id=t2.segment_head_id
where t1.flights_date>=to_date('2021-01-21','yyyy-mm-dd')
 and t1.flights_date<= to_date('2021-03-01','yyyy-mm-dd')
 and t1.company_id=0
 and t2.flag<>2
 and t1.order_day< to_date('2021-01-27','yyyy-mm-dd')
 and t1.seats_name is not null
 and t2.dest_country_id>0
 group by  t2.flights_segment_name,t1.segment_head_id,t2.oversale,t1.nationflag)h1
 group by year,flights_segment_name,nationflag;
````

#### 6 Provide passenger mobile phone number for SMS push

>Background:
In order to promote the new business of "Duty Free reservation" online, it is necessary to push marketing SMS and provide the passenger mobile phone number of marketing SMS. The requirements are: 
1. SMS push rules: push SMS on 20th, provide passenger information on 27th, push SMS on 21st, provide passenger information on 28th, and so on 
2. Flight information: 8876 Macau - Pudong, 8590 Osaka - Pudong, 8668 Phuket - Pudong, 8792 Sapporo - Pudong, 8522 Phuket - Pudong, 6244 Yangon - Pudong, 8878 Kaoxiong - Pudong, 8512 Chiang Mai - Pudong, 8516 Tokyo - Pudong, 6220 Sapporo - Pudong, 8980 Chiang Mai - Pudong, 8756 Bangkok - Pudong, 8550 Singapore - Pudong, 6218 Tokyo - Pudong, 8602 Nagoya - Pudong, 8594 Sabah - Pudong, 8560 Seoul - Pudong (total 17 flights) 
3. Passenger data: Provided on January 20: Passenger data from January 27 to February 7; January 31, with data available for February 8 - February 27

````sql
select r_flights_date,mobile
from(
select t1.r_flights_date,getmobile(t1.r_tel) mobile,row_number()over(partition by getmobile(t1.r_tel) order by t1.r_flights_date) rid
--distinct t2.nation_flag,t1.whole_segment
  from cqsale.cq_order@to_air t
  join cqsale.cq_order_head@to_air t1 on t.flights_order_id=t1.flights_order_id
  join cqsale.cq_flights_segment_head@to_air t2 on t1.segment_head_Id=t2.segment_head_Id
  where t1.r_flights_date>=to_date('2022-01-27','yyyy-mm-dd')
     and t1.r_flights_date<= to_date('2022-02-07','yyyy-mm-dd')
   and t1.flag_id in(3,5,40)
   and t1.whole_flight like '9C%'
   and t1.seats_name is not null
   and t1.sex=1
   and getmobile(t1.r_tel)<>'-'
   and t2.flag<>2
   and t1.whole_flight in('9C8876',
'9C8590',
'9C8668',
'9C8792',
'9C8522',
'9C6244',
'9C8878',
'9C8512',
'9C8516',
'9C6220',
'9C8980',
'9C8756',
'9C8550',
'9C6218',
'9C8602',
'9C8594',
'9C8560'))h1
where h1.rid=1;
````



#### 7 Refund during festival

>Background: Refund date data: 20200101~ now compared with 20190112~ The same date of the Spring Festival travel rush last year

````sql

 
 select '今年' 类型,trunc(t1.money_date) 退票日期,count(1)  退票量
from dw.da_order_drawback t1
join dw.da_flight t2 on t1.segment_head_id=t2.segment_head_id
where t1.money_date>=to_date('2022-01-01','yyyy-mm-dd')
  and t1.money_date< trunc(sysdate)
  and t2.flag<>2
  and t1.seats_name is not null
  group  by trunc(t1.money_date)

union all


select 'sametimelastyear' type,t3.datetime,count(1)
from dw.da_order_drawback t1
join dw.da_flight t2 on t1.segment_head_id=t2.segment_head_id
join (select * from dw.adt_chunyun_corredate 
where chunyun_year = 2020 and corre_year = 2019)t3 on trunc(t1.money_date)=t3.corre_date
where t1.money_date>=to_date('2021-01-12','yyyy-mm-dd')
  and t1.money_date< trunc(sysdate)
  and t3.datetime>=to_date('2021-01-01','yyyy-mm-dd')
  and t3.datetime< trunc(sysdate)
  and t2.flag<>2
  and t1.seats_name is not null
  group by t3.datetime;

````


#### 8 Refund during festival by Route

>Background: Segment refund volume gap

````sql

select h1.flights_segment_name Route, h1.ticketnum yesterdayrefound,nvl(h2.avgnum,0) "0113~0119averagerefound",
h1.ticketnum-nvl(h2.avgnum,0) ticketvolumedifference,row_number()over(order by h1.ticketnum-nvl(h2.avgnum,0) desc) ticketvolumeorder
from(
select t2.flights_segment_name,count(1) ticketnum
from dw.da_order_drawback t1
join dw.da_flight t2 on t1.segment_head_id=t2.segment_head_id
where t1.money_date>=trunc(sysdate-1)
  and t1.money_date< trunc(sysdate)
  and t2.flag<>2
  and t1.seats_name is not null
  group  by t2.flights_segment_name)h1
  left join (
  select flights_segment_name,round(avg(ticketnum),0) avgnum
  from(
  select t2.flights_segment_name,trunc(t1.money_date) moneydate,count(1) ticketnum
from dw.da_order_drawback t1
join dw.da_flight t2 on t1.segment_head_id=t2.segment_head_id
where t1.money_date>=to_date('2022-01-13','yyyy-mm-dd')
  and t1.money_date< to_date('2022-01-20','yyyy-mm-dd')
  and t2.flag<>2
  and t1.seats_name is not null
  group  by t2.flights_segment_name,trunc(t1.money_date))
  group by flights_segment_name)h2 on h1.flights_segment_name=h2.flights_segment_name;

````


#### 9 analysis about flights to Tailand

>Background: analysis the flights to Tailand, seat type, the customer data, the data from sametime lastyear.

````sql

SELECT /*+parallel(4) */
    CASE 
        WHEN t1.order_day >= TO_DATE('2020-02-14', 'yyyy-mm-dd') 
             AND t1.order_day <= TO_DATE('2020-02-20', 'yyyy-mm-dd') THEN 'Last Week'
        WHEN t1.order_day >= TO_DATE('2020-02-21', 'yyyy-mm-dd') 
             AND t1.order_day <= TO_DATE('2020-02-27', 'yyyy-mm-dd') THEN 'This Week'
    END AS Week_Type,
    '02' || TO_CHAR(t1.order_day, 'dd') AS Date,
    CASE 
        WHEN t2.flights_segment_name LIKE '%Bangkok%' THEN 'Bangkok'
        WHEN t2.flights_segment_name LIKE '%Phuket%' THEN 'Phuket'
        ELSE 'Other'
    END AS Terminal_Type,
    CASE 
        WHEN t2.origin_country_id > 0 THEN 'Inbound'
        ELSE 'Outbound'
    END AS Inbound_Outbound_Type,
    CASE 
        WHEN t1.IS_QU_HUI = 1 THEN 'Outbound'
        WHEN t1.IS_QU_HUI = 2 THEN 'Return'
        ELSE 'One-way'
    END AS Trip_Type,
    CASE 
        WHEN t1.IS_QU_HUI IN (1, 2) AND t6.flights_order_head_id IS NOT NULL THEN ABS(t1.flights_date - t6.flights_date)
        ELSE NULL
    END AS Round_Trip_Days,
    CASE 
        WHEN t1.channel IN ('Website', 'Mobile') THEN 'Online - Direct'
        WHEN t1.channel IN ('OTA', 'Flagship Store') THEN 'OTA'
        ELSE 'Offline - Other'
    END AS Channel,
    CASE
        WHEN t1.ahead_days <= 3 THEN '00~03'
        WHEN t1.ahead_days <= 7 THEN '04~07'
        WHEN t1.ahead_days <= 15 THEN '08~15'
        WHEN t1.ahead_days <= 30 THEN '16~30'
        WHEN t1.ahead_days <= 60 THEN '31~60'
        WHEN t1.ahead_days <= 90 THEN '61~90'
        ELSE '90+'
    END AS Booking_Lead_Time,
    t1.seat_type AS Seat_Type,
    t3.wf_segment AS Return_Segment,
    CASE 
        WHEN t5.nationality LIKE '%China%' THEN 'China'
        WHEN t5.nationality LIKE '%Thailand%' THEN 'Thailand'
        ELSE 'Other'
    END AS Nationality,
    t1.gender AS Gender,
    CASE 
        WHEN getage(t1.flights_date, t1.birthday) < 12 THEN '<12'
        WHEN getage(t1.flights_date, t1.birthday) < 18 THEN '12~17'
        WHEN getage(t1.flights_date, t1.birthday) < 24 THEN '18~23'
        WHEN getage(t1.flights_date, t1.birthday) < 30 THEN '24~29'
        WHEN getage(t1.flights_date, t1.birthday) < 40 THEN '30~39'
        WHEN getage(t1.flights_date, t1.birthday) < 50 THEN '40~49'
        WHEN getage(t1.flights_date, t1.birthday) < 60 THEN '50~59'
        WHEN getage(t1.flights_date, t1.birthday) < 70 THEN '60~69'
        ELSE '70+'
    END AS Age_Group,
    COUNT(1) AS Sales_Volume,
    SUM(t1.ticket_price) AS Ticket_Revenue
FROM 
    dw.fact_recent_order_detail t1
JOIN 
    dw.da_flight t2 ON t1.segment_head_id = t2.segment_head_id
LEFT JOIN 
    dw.fact_recent_order_detail t6 ON t1.WF_FLIGHTS_ORDER_HEAD_ID = t6.flights_order_head_id
LEFT JOIN 
    dw.dim_segment_type t3 ON t2.h_route_id = t3.h_route_id AND t2.route_id = t3.route_id
LEFT JOIN 
    dw.bi_order_region t5 ON t1.flights_order_head_id = t5.flights_order_head_id
WHERE 
    t1.order_day >= TO_DATE('2020-02-14', 'yyyy-mm-dd')
    AND t1.order_day <= TO_DATE('2020-02-27', 'yyyy-mm-dd')
    AND t1.whole_flight LIKE '9C%'
    AND t1.flag_id IN (3, 5, 40, 41)
    AND t2.flag <> 2
    AND t1.seats_name IS NOT NULL
    AND t2.segment_country = 'Thailand'
    AND t1.seats_name NOT IN ('B', 'G', 'G1', 'G2', 'O')
GROUP BY 
    CASE 
        WHEN t1.order_day >= TO_DATE('2020-02-14', 'yyyy-mm-dd') 
             AND t1.order_day <= TO_DATE('2020-02-20', 'yyyy-mm-dd') THEN 'Last Week'
        WHEN t1.order_day >= TO_DATE('2020-02-21', 'yyyy-mm-dd') 
             AND t1.order_day <= TO_DATE('2020-02-27', 'yyyy-mm-dd') THEN 'This Week'
    END,
    CASE 
        WHEN t2.flights_segment_name LIKE '%Bangkok%' THEN 'Bangkok'
        WHEN t2.flights_segment_name LIKE '%Phuket%' THEN 'Phuket'
        ELSE 'Other'
    END,
    CASE 
        WHEN t2.origin_country_id > 0 THEN 'Inbound'
        ELSE 'Outbound'
    END,
    CASE 
        WHEN t1.IS_QU_HUI = 1 THEN 'Outbound'
        WHEN t1.IS_QU_HUI = 2 THEN 'Return'
        ELSE 'One-way'
    END,
    CASE 
        WHEN t1.IS_QU_HUI IN (1, 2) AND t6.flights_order_head_id IS NOT NULL THEN ABS(t1.flights_date - t6.flights_date)
        ELSE NULL
    END,
    CASE 
        WHEN t1.channel IN ('Website', 'Mobile') THEN 'Online - Direct'
        WHEN t1.channel IN ('OTA', 'Flagship Store') THEN 'OTA'
        ELSE 'Offline - Other'
    END,
    CASE
        WHEN t1.ahead_days <= 3 THEN '00~03'
        WHEN t1.ahead_days <= 7 THEN '04~07'
        WHEN t1.ahead_days <= 15 THEN '08~15'
        WHEN t1.ahead_days <= 30 THEN '16~30'
        WHEN t1.ahead_days <= 60 THEN '31~60'
        WHEN t1.ahead_days <= 90 THEN '61~90'
        ELSE '90+'
    END,
    t1.seat_type,
    t3.wf_segment,
    CASE 
        WHEN t5.nationality LIKE '%China%' THEN 'China'
        WHEN t5.nationality LIKE '%Thailand%' THEN 'Thailand'
        ELSE 'Other'
    END,
    t1.gender,
    CASE 
        WHEN getage(t1.flights_date, t1.birthday) < 12 THEN '<12'
        WHEN getage(t1.flights_date, t1.birthday) < 18 THEN '12~17'
        WHEN getage(t1.flights_date, t1.birthday) < 24 THEN '18~23'
        WHEN getage(t1.flights_date, t1.birthday) < 30 THEN '24~29'
        WHEN getage(t1.flights_date, t1.birthday) < 40 THEN '30~39'
        WHEN getage(t1.flights_date, t1.birthday) < 50 THEN '40~49'
        WHEN getage(t1.flights_date, t1.birthday) < 60 THEN '50~59'
        WHEN getage(t1.flights_date, t1.birthday) < 70 THEN '60~69'
        ELSE '70+'
    END,
    '02' || TO_CHAR(t1.order_day, 'dd')


      
union all



SELECT 
    CASE 
        WHEN t1.order_day >= TO_DATE('2019-02-15', 'yyyy-mm-dd') 
             AND t1.order_day <= TO_DATE('2019-02-21', 'yyyy-mm-dd') THEN 'Last Year - Last Week'
        WHEN t1.order_day >= TO_DATE('2019-02-22', 'yyyy-mm-dd') 
             AND t1.order_day <= TO_DATE('2019-02-28', 'yyyy-mm-dd') THEN 'Last Year - This Week'
    END AS Week_Type, 
    '02' || TO_CHAR(TO_NUMBER(TO_CHAR(t1.order_day, 'dd')) - 1) AS Date,
    CASE 
        WHEN t2.flights_segment_name LIKE '%Bangkok%' THEN 'Bangkok'
        WHEN t2.flights_segment_name LIKE '%Phuket%' THEN 'Phuket'
        ELSE 'Other'
    END AS Terminal_Type,
    CASE 
        WHEN t2.origin_country_id > 0 THEN 'Inbound'
        ELSE 'Outbound'
    END AS Inbound_Outbound_Type,
    CASE 
        WHEN t1.IS_QU_HUI = 1 THEN 'Outbound'
        WHEN t1.IS_QU_HUI = 2 THEN 'Return'
        ELSE 'One-way'
    END AS Trip_Type,
    CASE 
        WHEN t1.IS_QU_HUI IN (1, 2) 
             AND t6.flights_order_head_id IS NOT NULL THEN ABS(t1.flights_date - t6.flights_date)
        ELSE NULL
    END AS Round_Trip_Days,
    CASE 
        WHEN t1.channel IN ('Website', 'Mobile') THEN 'Online - Direct'
        WHEN t1.channel IN ('OTA', 'Flagship Store') THEN 'OTA'
        ELSE 'Offline - Other'
    END AS Channel,
    CASE
        WHEN t1.ahead_days <= 3 THEN '00~03'
        WHEN t1.ahead_days <= 7 THEN '04~07'
        WHEN t1.ahead_days <= 15 THEN '08~15'
        WHEN t1.ahead_days <= 30 THEN '16~30'
        WHEN t1.ahead_days <= 60 THEN '31~60'
        WHEN t1.ahead_days <= 90 THEN '61~90'
        ELSE '90+'
    END AS Booking_Lead_Time,
    t1.seat_type AS Seat_Type,
    t3.wf_segment AS Return_Segment,
    CASE 
        WHEN t5.nationality LIKE '%China%' THEN 'China'
        WHEN t5.nationality LIKE '%Thailand%' THEN 'Thailand'
        ELSE 'Other'
    END AS Nationality,
    t1.gender AS Gender,
    CASE 
        WHEN getage(t1.flights_date, t1.birthday) < 12 THEN '<12'
        WHEN getage(t1.flights_date, t1.birthday) < 18 THEN '12~17'
        WHEN getage(t1.flights_date, t1.birthday) < 24 THEN '18~23'
        WHEN getage(t1.flights_date, t1.birthday) < 30 THEN '24~29'
        WHEN getage(t1.flights_date, t1.birthday) < 40 THEN '30~39'
        WHEN getage(t1.flights_date, t1.birthday) < 50 THEN '40~49'
        WHEN getage(t1.flights_date, t1.birthday) < 60 THEN '50~59'
        WHEN getage(t1.flights_date, t1.birthday) < 70 THEN '60~69'
        ELSE '70+'
    END AS Age_Group,
    COUNT(1) AS Sales_Volume,
    SUM(t1.ticket_price) AS Ticket_Revenue
FROM 
    dw.fact_recent_order_detail t1
JOIN 
    dw.da_flight t2 ON t1.segment_head_id = t2.segment_head_id
LEFT JOIN 
    dw.fact_recent_order_detail t6 ON t1.WF_FLIGHTS_ORDER_HEAD_ID = t6.flights_order_head_id
LEFT JOIN 
    dw.dim_segment_type t3 ON t2.h_route_id = t3.h_route_id AND t2.route_id = t3.route_id
LEFT JOIN 
    dw.bi_order_region t5 ON t1.flights_order_head_id = t5.flights_order_head_id
WHERE 
    t1.order_day >= TO_DATE('2019-02-15', 'yyyy-mm-dd')
    AND t1.order_day <= TO_DATE('2019-02-28', 'yyyy-mm-dd')
    AND t1.whole_flight LIKE '9C%'
    AND t1.flag_id IN (3, 5, 40, 41)
    AND t2.flag <> 2
    AND t1.seats_name IS NOT NULL
    AND t2.segment_country = 'Thailand'
    AND t1.seats_name NOT IN ('B', 'G', 'G1', 'G2', 'O')
GROUP BY 
    CASE 
        WHEN t1.order_day >= TO_DATE('2019-02-15', 'yyyy-mm-dd') 
             AND t1.order_day <= TO_DATE('2019-02-21', 'yyyy-mm-dd') THEN 'Last Year - Last Week'
        WHEN t1.order_day >= TO_DATE('2019-02-22', 'yyyy-mm-dd') 
             AND t1.order_day <= TO_DATE('2019-02-28', 'yyyy-mm-dd') THEN 'Last Year - This Week'
    END,
    '02' || TO_CHAR(TO_NUMBER(TO_CHAR(t1.order_day, 'dd')) - 1),
    CASE 
        WHEN t2.flights_segment_name LIKE '%Bangkok%' THEN 'Bangkok'
        WHEN t2.flights_segment_name LIKE '%Phuket%' THEN 'Phuket'
        ELSE 'Other'
    END,
    CASE 
        WHEN t2.origin_country_id > 0 THEN 'Inbound'
        ELSE 'Outbound'
    END,
    CASE 
        WHEN t1.IS_QU_HUI = 1 THEN 'Outbound'
        WHEN t1.IS_QU_HUI = 2 THEN 'Return'
        ELSE 'One-way'
    END,
    CASE 
        WHEN t1.IS_QU_HUI IN (1, 2) 
             AND t6.flights_order_head_id IS NOT NULL THEN ABS(t1.flights_date - t6.flights_date)
        ELSE NULL
    END,
    CASE 
        WHEN t1.channel IN ('Website', 'Mobile') THEN 'Online - Direct'
        WHEN t1.channel IN ('OTA', 'Flagship Store') THEN 'OTA'
        ELSE 'Offline - Other'
    END,
    CASE
        WHEN t1.ahead_days <= 3 THEN '00~03'
        WHEN t1.ahead_days <= 7 THEN '04~07'
        WHEN t1.ahead_days <= 15 THEN '08~15'
        WHEN t1.ahead_days <= 30 THEN '16~30'
        WHEN t1.ahead_days <= 60 THEN '31~60'
        WHEN t1.ahead_days <= 90 THEN '61~90'
        ELSE '90+'
    END,
    t1.seat_type,
    t3.wf_segment,
    CASE 
        WHEN t5.nationality LIKE '%China%' THEN 'China'
        WHEN t5.nationality LIKE '%Thailand%' THEN 'Thailand'
        ELSE 'Other'
    END,
    t1.gender,
    CASE 
        WHEN getage(t1.flights_date, t1.birthday) < 12 THEN '<12'
        WHEN getage(t1.flights_date, t1.birthday) < 18 THEN '12~17'
        WHEN getage(t1.flights_date, t1.birthday) < 24 THEN '18~23'
        WHEN getage(t1.flights_date, t1.birthday) < 30 THEN '24~29'
        WHEN getage(t1.flights_date, t1.birthday) < 40 THEN '30~39'
        WHEN getage(t1.flights_date, t1.birthday) < 50 THEN '40~49'
        WHEN getage(t1.flights_date, t1.birthday) < 60 THEN '50~59'
        WHEN getage(t1.flights_date, t1.birthday) < 70 THEN '60~69'
        ELSE '70+'
    END

````




#### 10 Japanese currency payment data


````sql
 select t1.flights_date,t1.whole_flight,t1.channel,t1.sub_channel,count(1)
  from dw.fact_order_detail t1
  where t1.whole_flight in('9C8889','9C8890')
  and t1.flights_date>=to_date('2021-10-01','yyyy-mm-dd')
  and t1.flights_date< to_date('2022-03-01','yyyy-mm-dd')
  and t1.money_class_id=1
  and t1.seats_name is not null
  group by t1.flights_date,t1.whole_flight,t1.channel,t1.sub_channel
````


#### 11 Statistics of international passenger load factor in East China


````sql
select /*+parallel(4) */
tb3.flight_date ,tb3.flight_no,
tt1.c_name||'-'||tt2.c_name route,
tb3.origin_country origin,
tb3.dest_country destiny,
tb3.layout seats,
nvl(ticketnum,0) appointments
from  cqsale.cq_flights_segment_head@to_air tb1 
join cqsale.cq_flights_seats_amount_plan@to_air t6  on tb1.segment_Head_id=t6.segment_Head_id
join dw.da_flight tb3  on tb1.segment_head_id=tb3.segment_head_id
left join stg.f_airport tt1 on tb3.originairport=tt1.t_code
left join stg.f_airport tt2 on tb3.destairport=tt2.t_code
left join 
(
select t1.segment_head_id,count(1)  ticketnum
from cqsale.cq_order_head@to_air t1
join dw.da_flight t2 on t1.segment_head_id=t2.segment_head_id
where t1.r_flights_date>=trunc(sysdate)
and t1.r_flights_date<=trunc(sysdate)+6
and t1.flag_id in(3,5,40,41)
and t2.company_id=0
and t2.flag<>2
and t1.seats_name is not null
and t2.flights_segment_name like '%Pudong%'
and t2.nationflag<>'inland'
group by t1.segment_head_id)tb4 on tb3.segment_head_id=tb4.segment_head_id
where tb1.flag<>2
and  tb3.flight_date>=trunc(sysdate)
and  tb3.flight_date<=trunc(sysdate)+6
and tb3.company_id=0
and tb3.flights_segment_name like '%Pudong%'
and tb3.nationflag<>'inland'
order by 1,2
````


#### 12 Passenger information for entering Ningbo

> Ningbo Airport wants a data, from international to domestic, domestic passenger information into Ningbo


````sql
SELECT 
    t1.flights_date_1 AS First_Segment_Flight_Date,
    t1.flights_no_1 AS First_Segment_Flight_Number,
    t1.flights_segment_1 AS First_Segment,
    t1.flights_date_2 AS Second_Segment_Flight_Date,
    t1.flight_no_2 AS Second_Segment_Flight_Number,
    t1.flights_segment_2 AS Second_Segment,
    t1.en_name AS Name,
    t4.codetype_name AS Document_Type,
    t1.codeno AS Document_Number,
    t2.r_tel AS Emergency_Contact,
    t2.work_tel AS Booker_Contact
FROM 
    dw.bi_connect_segment t1
LEFT JOIN 
    dw.fact_order_detail t2 ON t1.flights_order_head_id_1 = t2.flights_order_head_id
LEFT JOIN 
    dw.fact_order_detail t3 ON t1.flights_order_head_id_2 = t3.flights_order_head_id
LEFT JOIN 
    stg.s_cq_codetype t4 ON t1.codetype = t4.codetype
WHERE 
    t1.flights_segment_2 LIKE '%NGB'
    AND t1.flights_date_2 <= TO_DATE('2022-03-16', 'yyyy-mm-dd')
    AND t1.flights_date_2 >= TO_DATE('2022-03-09', 'yyyy-mm-dd')
    AND t1.nationflag_1 <> 'Domestic'
    AND t1.nationflag_2 = 'Domestic';

````


#### 13 SMS to certain group of customer

> In the whole year of 2019, the number of female passengers aged 28-35 who took flights more than four times (including four times) is estimated to be about 40,000


````sql
  select /*+parallel(4) */
distinct r_tel
 from(
 select t1.traveller_name||t1.codeno,t1.r_tel,count(1) ticketnum,sum(count(1))over(partition by t1.traveller_name||t1.codeno) accnum
  from dw.fact_order_detail t1
 where t1.flights_date >= to_date('2021-01-01','yyyy-mm-dd')
   and t1.flights_date < to_date('2022-01-01','yyyy-mm-dd')
   and t1.company_id = 0
   and t1.gender='女'
   and t1.flag_id=40
   and t1.seats_name <>'O'
   and getage(t1.flights_date,t1.birthday)>=28
   and getage(t1.flights_date,t1.birthday)<=35
   group by t1.traveller_name||t1.codeno,t1.r_tel)
   where accnum>=4
   and getmobile(r_tel)<>'-'
   and rownum<=40000

````

#### 14 Freight data entry


````sql

 insert into stg.wb_freight_charter
select distinct t1.flight_no,t1.flight_date,t1.flights_segment_name,192000/2 income,sysdate
 from dw.da_flight t1
 where t1.flag in(0,1)
 and t1.flight_no in('9C6239','9C6240')
 and t1.flight_date=to_date('2020-04-28','yyyy-mm-dd')
 
 union all
 
 select distinct t1.flight_no,t1.flight_date,t1.flights_segment_name,220000/2 income,sysdate
 from dw.da_flight t1
 where t1.flag in(0,1)
 and t1.flight_no in('9C6173','9C6174')
 and t1.flight_date=to_date('2020-04-28','yyyy-mm-dd')
 
 union all
 

select distinct t1.flight_no,t1.flight_date,t1.flights_segment_name,188000/2 income,sysdate
 from dw.da_flight t1
 where t1.flag in(0,1)
 and t1.flight_no in('9C8999','9C9000')
 and t1.flight_date=to_date('2020-04-28','yyyy-mm-dd')
 
union all

select distinct t1.flight_no,t1.flight_date,t1.flights_segment_name,188000/2,sysdate
 from dw.da_flight t1
 where t1.flag in(0,1)
 and t1.flight_no in('9C8589','9C8590')
and t1.flight_date=to_date('2020-04-28','yyyy-mm-dd')
````

#### 15 total flights in five years

> total flights in five years

1, in the past five years, the total number of passengers flying to Xinjiang each year, winter and spring and summer and autumn are separated. 
2. In the past five years, the number of tourists we fly to Xinjiang every year has been separated from winter, spring and summer and autumn. 
3, the above data need to enter and leave Xinjiang common data.

 
select *
 from dw.fact_order_detail t1
 join dw.da_flight t2 on t1.segment_Head_id=t2.segment_Head_id
 join dw.da_flight_season t3 on t2.flight_date>=t3.
 where t1.flights_date>=to_date('2015-03-29','yyyy-mm-dd')
    and t1.flights_date< to_date('2020-05-03','yyyy-mm-dd')
  and t1.flag<>2
  and regexp_like(t2.flights_segment_name,'(Urumqi)|(Karamay)|(Shache)')
  











