# Data Model

The Power BI model is a star schema with two fact tables sharing three dimension tables.

## Tables

| Table | Type | Rows | Grain |
|---|---|---|---|
| `fact_bookings` | Fact | 134,590 | One row per booking |
| `fact_aggregated_bookings` | Fact | 9,200 | One row per hotel, room type and check-in date |
| `dim_date` | Dimension | 92 | One row per day, 1 May to 31 July 2022 |
| `dim_hotels` | Dimension | 25 | One row per hotel |
| `dim_rooms` | Dimension | 4 | One row per room type |
| `_Measures` | Measures table | 0 | Holds all DAX measures |

## Relationships

All relationships are many-to-one with single-direction filtering.

| From (many) | To (one) |
|---|---|
| `fact_bookings[property_id]` | `dim_hotels[property_id]` |
| `fact_bookings[room_category]` | `dim_rooms[room_id]` |
| `fact_bookings[check_in_date]` | `dim_date[date]` |
| `fact_aggregated_bookings[property_id]` | `dim_hotels[property_id]` |
| `fact_aggregated_bookings[room_category]` | `dim_rooms[room_id]` |
| `fact_aggregated_bookings[check_in_date]` | `dim_date[date]` |

## Power Query changes

**dim_date**
- Removed `day_type`. The raw column treats Saturday and Sunday as the weekend. In the hotel industry the busy weekend nights are Friday and Saturday, so it was rebuilt as a calculated column.
- Removed `week no`. It stored text such as `W 19` in every row, which cannot be sorted or used in calculations. It was rebuilt as a number in the calculated column `wn`.
- Set `mmm yy` to a date type. Each value becomes the first day of its month, which is used for the month slicer and trend chart.

**All other tables**
- Promoted headers and set data types. No rows were removed.

## Report-level filter

- `dim_date[wn]` is not 32. This removes Sunday 31 July, the only day in week 32, from every visual.

## Calculated columns

All three are in `dim_date`.

```dax
wn = WEEKNUM(dim_date[date])
```
Week number, with weeks starting on Sunday. This matches the raw `week no` values.

```dax
day_type =
VAR wkd = WEEKDAY(dim_date[date])
RETURN IF(wkd > 5, "Weekend", "Weekday")
```
`WEEKDAY` returns 1 for Sunday through 7 for Saturday, so values above 5 are Friday (6) and Saturday (7). This gives 26 weekend days: 13 Fridays and 13 Saturdays.

```dax
year = YEAR(dim_date[date])
```

## Things to know about the data

- **Two hotels share a name.** Two different properties are called *Atliq Exotica* in Mumbai (IDs 16559 and 17559). Always use `property_id` to tell hotels apart.
- **`successful_bookings` includes cancelled bookings.** It equals the total number of bookings for each hotel, room type and day. As a result, **Occupancy %** counts cancelled rooms as occupied. This follows the challenge's metric definition.
- **Week 32 has one day.** The data ends on Sunday 31 July, which starts a new week. A report-level filter excludes week 32 so the report covers 13 complete weeks (1 May to 30 July). See [`dax_measures.md`](dax_measures.md).
- **Cancelled bookings keep 40% of their revenue** as a cancellation fee. No-show bookings are charged in full.

## Suggested model settings

- Turn off **Auto date/time** (File, Options, Data Load). The model currently has five hidden auto-generated date tables that are not needed because `dim_date` exists.
- Mark `dim_date` as the date table (Table tools, Mark as date table).
