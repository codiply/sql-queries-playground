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
) as this
left join flights as previous 
  on this.previous_arrival_flight_no = previous.flight_no
  and date_trunc('day', this.actual_arrival) = date_trunc('day', previous.actual_arrival)
limit 1000
```