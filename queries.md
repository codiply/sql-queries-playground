# Queries

## Window Functions

Find the previous arrival at a specific airport

```
SELECT this.flight_no
     , this.actual_arrival
     , this.arrival_airport
     , previous.flight_no AS previous_flight_no
     , previous.actual_arrival AS previous_actual_arrival
     , previous.arrival_airport AS previous_arrival_airport
FROM
(
SELECT LAG(flight_no) OVER (
             PARTITION BY arrival_airport, date_trunc('day', actual_arrival)
ORDER BY actual_arrival
          ) AS previous_arrival_flight_no
     , *
FROM flights
WHERE actual_arrival IS NOT NULL
AND arrival_airport = 'DME'
) AS this
LEFT JOIN flights AS previous 
  ON
this.previous_arrival_flight_no = previous.flight_no
AND date_trunc('day', this.actual_arrival) = date_trunc('day', previous.actual_arrival)
LIMIT 1000 a
```

Find flights that have at least 5 empty rows between 2 non-empty rows

```
SELECT DISTINCT flight_id
FROM (
  SELECT flight_id
       , seat_row
       , LAG(seat_row) OVER (
           PARTITION BY flight_id
           ORDER BY seat_row
         ) AS previous_seat_row
FROM (
    SELECT *
         , CAST(substring(seat_no, 1, length(seat_no) - 1) AS integer) AS seat_row
FROM boarding_passes
  ) AS t1
) AS t2
WHERE previous_seat_row IS NOT NULL
AND seat_row - previous_seat_row - 1 >= 5
```

Find flights that have at least 5 empty rows between 2 full rows

```
WITH all_seats_with_tickets 
AS (
  SELECT *
       , cast(substring(seat_no, 1, length(seat_no) - 1) as integer) as seat_row
  FROM (
    SELECT q1.flight_id 
         , q1.seat_no
         , bp.ticket_no
    FROM (
      SELECT
         fl.flight_id 
       , s.seat_no
      FROM flights AS fl
      JOIN seats AS s 
        ON s.aircraft_code = fl.aircraft_code
    ) AS q1
    LEFT JOIN boarding_passes as bp 
      ON q1.flight_id = bp.flight_id  
      AND bp.seat_no = q1.seat_no
  ) AS q2
), row_full_pct AS (
  SELECT flight_id
       , seat_row
       , CAST (count(ticket_no) AS float) / count(*) AS pct_full
  FROM all_seats_with_tickets
  GROUP BY flight_id, seat_row
), non_empty_rows AS (
  SELECT *
  FROM row_full_pct
  WHERE pct_full > 0.0
), row_with_previous_row AS (
  SELECT flight_id
       , seat_row
       , pct_full
       , LAG(seat_row) OVER (
           PARTITION BY flight_id 
           ORDER BY seat_row
         ) AS previous_seat_row
       , LAG(pct_full) OVER (
           PARTITION BY flight_id 
           ORDER BY seat_row
         ) AS previous_pct_full
  FROM non_empty_rows
)
SELECT *
FROM row_with_previous_row
WHERE seat_row - previous_seat_row - 1 >= 5
  AND pct_full = 1.0
  AND previous_pct_full = 1.0
```

Find all groups of 3 passengers that
- Have boarded consecutively
- They are seated on the same row next to each other
- The order of boarding can be different from the order they are seated

```
WITH all_seats AS
(
SELECT aircraft_code 
     , seat_no
     , substring(seat_no, 1, length(seat_no) - 1)::integer AS row_no
     , substring(seat_no, length(seat_no), 1) AS seat_letter
FROM seats
), joined_data AS
(
SELECT flights.flight_id 
     , boarding_passes.seat_no
     , boarding_passes.boarding_no 
     , all_seats.row_no
     , all_seats.seat_letter
     , all_seats.aircraft_code
FROM flights
LEFT JOIN all_seats
  ON all_seats.aircraft_code = flights.aircraft_code
LEFT JOIN boarding_passes
  ON boarding_passes.flight_id = flights.flight_id 
  AND boarding_passes.seat_no = all_seats.seat_no
WHERE flights.flight_id < 100 -- Remove to run query for all flights 
), windows AS 
(
SELECT * 
     , count(boarding_no) OVER w AS passenger_count
     , min(boarding_no) OVER w AS minimum_boarding_number
     , max(boarding_no) OVER w AS maximum_boarding_number
FROM joined_data
WINDOW w AS (PARTITION BY flight_id, row_no ORDER BY seat_letter ROWS BETWEEN CURRENT ROW AND 2 FOLLOWING)
), first_passenger_poarding AS
(
SELECT flight_id
     , aircraft_code 
     , minimum_boarding_number AS first_passenger_boarding_no
     , ROW_NUMBER() OVER(PARTITION BY flight_id ORDER BY boarding_no) AS group_id
FROM windows
WHERE passenger_count = 3
  AND maximum_boarding_number - minimum_boarding_number = 2
), passenger_groups AS (
SELECT flight_id
     , aircraft_code 
     , first_passenger_boarding_no AS boarding_no
     , group_id
FROM first_passenger_poarding
UNION ALL
SELECT flight_id
     , aircraft_code 
     , first_passenger_boarding_no + 1 AS boarding_no
     , group_id
FROM first_passenger_poarding
UNION ALL 
SELECT flight_id
     , aircraft_code 
     , first_passenger_boarding_no + 2 AS boarding_no
     , group_id
FROM first_passenger_poarding
)
SELECT passenger_groups.flight_id
     , passenger_groups.group_id
     , passenger_groups.boarding_no
     , boarding_passes.seat_no
     , passenger_groups.aircraft_code 
FROM passenger_groups
LEFT JOIN boarding_passes
  ON passenger_groups.boarding_no = boarding_passes.boarding_no
  AND passenger_groups.flight_id = boarding_passes.flight_id 
ORDER BY passenger_groups.flight_id
       , passenger_groups.group_id
       , passenger_groups.boarding_no
```