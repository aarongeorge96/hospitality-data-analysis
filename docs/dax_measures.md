# DAX Measures

All measures are stored in the `_Measures` table. Definitions follow the metrics list provided with the challenge (`data/raw/metrics-list.xlsx`).

## Base measures

| Measure | DAX | Meaning |
|---|---|---|
| Revenue | `SUM(fact_bookings[revenue_realized])` | Revenue actually collected |
| Total Bookings | `COUNT(fact_bookings[booking_id])` | All bookings, any status |
| Total Capacity | `SUM(fact_aggregated_bookings[capacity])` | Room nights available to sell |
| Total Succesful Bookings | `SUM(fact_aggregated_bookings[successful_bookings])` | Room nights booked |
| Average Rating | `AVERAGE(fact_bookings[ratings_given])` | Average guest rating, 1 to 5 |
| No of days | `DATEDIFF(MIN(dim_date[date]), MAX(dim_date[date]), DAY) + 1` | Days in the selected period |

## Booking status measures

```dax
Total cancelled bookings = CALCULATE([Total Bookings], fact_bookings[booking_status] = "Cancelled")
Total Checked Out        = CALCULATE([Total Bookings], fact_bookings[booking_status] = "Checked Out")
Total no show bookings   = CALCULATE([Total Bookings], fact_bookings[booking_status] = "No Show")

Cancellation %  = DIVIDE([Total cancelled bookings], [Total Bookings], 0)
No Show rate %  = DIVIDE([Total no show bookings], [Total Bookings], 0)
Realisation %   = 1 - ([No Show rate %] + [Cancellation %])
```

## Hotel performance measures

```dax
Occupancy % = DIVIDE([Total Succesful Bookings], [Total Capacity], 0)
ADR         = DIVIDE([Revenue], [Total Bookings])
RevPAR      = DIVIDE([Revenue], [Total Capacity])
DBRN        = DIVIDE([Total Bookings], [No of days], 0)
DSRN        = DIVIDE([Total Capacity], [No of days])
DURN        = DIVIDE([Total Checked Out], [No of days], 0)
```

| Measure | Full name | Meaning |
|---|---|---|
| ADR | Average Daily Rate | Average revenue per booking |
| RevPAR | Revenue Per Available Room | Revenue divided by all rooms available, booked or not |
| DBRN | Daily Booked Room Nights | Average rooms booked per day |
| DSRN | Daily Sellable Room Nights | Average rooms available per day |
| DURN | Daily Utilized Room Nights | Average rooms actually used per day |

## Share measures

Formatted as percentages in the report.

```dax
Booking % by Platform =
DIVIDE([Total Bookings], CALCULATE([Total Bookings], ALL(fact_bookings[booking_platform])))

Booking % by Room class =
DIVIDE([Total Bookings], CALCULATE([Total Bookings], ALL(dim_rooms[room_class])))
```

## Week-over-week measures

Each card on the dashboard shows the change from the previous week. All six measures follow the same pattern. Revenue is shown here:

```dax
Revenue WoW change % =
VAR selv =
    IF(
        HASONEVALUE(dim_date[wn]),
        VALUES(dim_date[wn]),
        MAX(dim_date[wn])
    )
VAR revcw = CALCULATE([Revenue], REMOVEFILTERS(dim_date), dim_date[wn] = selv)
VAR revpw = CALCULATE([Revenue], REMOVEFILTERS(dim_date), dim_date[wn] = selv - 1)
RETURN
    DIVIDE(revcw - revpw, revpw)
```

When one week is selected, the measure compares it with the week before. When no week or several weeks are selected, it uses the latest week. The other five measures swap `[Revenue]` for `[Occupancy %]`, `[ADR]`, `[RevPAR]`, `[Realisation %]` or `[DSRN]`.

### Fixes made during validation

1. **Occupancy WoW change % calculated revenue.** The measure used `[Revenue]` in both variables, so the Occupancy card showed the revenue change. It now uses `[Occupancy %]`.
2. **The default comparison used a one-day week.** Week 32 contains only Sunday 31 July. With no week selected, the cards compared that single day against all of week 31 and showed revenue falling by 81.7%. A report-level filter now excludes week 32, so the latest week is week 31 and the cards show the real change of +0.2%.
