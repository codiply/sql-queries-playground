# Queries

## Window Functions

Find the previous arrival at a specific airport

```
select this.flight_no
     , this.actual_arrival
     , this.arrival_airport
     , previous.flight_no as previous_flight_no
     , previous.actual_arrival as previous_actual_arrival
     , previous.arrival_airport as previous_arrival_airport
from
(
select lag(flight_no) over (
             partition by arrival_airport, date_trunc('day', actual_arrival) 
             order by actual_arrival
          ) as previous_arrival_flight_no
     , *
from flights
where actual_arrival is not null
  and arrival_airport = 'DME'
) as this
left join flights as previous 
  on this.previous_arrival_flight_no = previous.flight_no
  and date_trunc('day', this.actual_arrival) = date_trunc('day', previous.actual_arrival)
limit 1000
```

Find flights that have at least 5 empty rows between 2 full rows

```
select distinct flight_id
from (
  select flight_id
       , seat_row
       , lag(seat_row) over (
           partition by flight_id order by seat_row
         ) as previous_seat_row
  from (
    select *
         , cast(substring(seat_no, 1, length(seat_no) - 1) as integer) as seat_row
    from boarding_passes
  ) as t1
) as t2
where previous_seat_row is not null 
  and seat_row - previous_seat_row - 1 >= 5
```