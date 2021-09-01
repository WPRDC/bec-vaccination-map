# Notes

## SQL query backing the map
There are some steps missing between the data extaction and getting to this point, but hopefully this get across how the data was generated.

```sql
SELECT *
FROM (SELECT *,
             -- percents
             TRUNC((100 * fully_covered_african_american::numeric / black_eligible::numeric), 1) as pct_black_covered,
             TRUNC((100 * fully_covered_total::numeric / total_eligible::numeric), 1)            as pct_covered
      FROM (SELECT z.cartodb_id,
                   z.zip_code                                    as geoid,
                   z.zip_code                                    as zip_code,

                   z.the_geom,
                   z.the_geom_webmercator,

                   z.black_pop                                   as black_pop,
                   z.black_kid_pop * 1.2                         as black_kid_pop,
                   z.black_pop - (z.black_kid_pop * 1.2)         as black_eligible,
                   z.black_pop_margin                            as black_pop_margin,
                   z.black_kid_pop_margin                        as black_kid_pop_margin,

                   z.total_pop                                   as total_pop,
                   z.kid_pop * 1.2                               as total_kid_pop,
                   z.total_pop - (z.kid_pop * 1.2)               as total_eligible,
                   z.total_pop_margin                            as total_pop_margin,
                   z.kid_pop_margin                              as total_kid_pop_margin,


                   COALESCE(v.fully_covered_african_american, 0) as fully_covered_african_american,
                   COALESCE(v.fully_covered_total, 0)            as fully_covered_total,
                   v.patient_zip_code

            FROM (
                     SELECT *,
                            (COALESCE(fully_covered_african_american, 0)
                                + COALESCE(fully_covered_asian, 0)
                                + COALESCE(fully_covered_native_american, 0)
                                + COALESCE(fully_covered_pacific_islander, 0)
                                + COALESCE(fully_covered_multiple_other, 0)
                                + COALESCE(fully_covered_white, 0)
                                + COALESCE(fully_covered_unknown, 0)) as fully_covered_total
                     FROM wprdc.covid_19_vaccinations_by_zip_code_by_race_current_health_1
                 ) v
                     JOIN zip_code_vaccine_pop z
                          ON v.patient_zip_code::text = z.zip_code

            WHERE z.black_pop > 0) as ssq
      WHERE black_eligible > 0
        AND total_eligible > 0) as sq
WHERE pct_black_covered < 200
```


## Data collection script
This uses [datamade's census api wrapper](https://github.com/datamade/census/tree/master/census) to download the data we want for all the zipcodes in PA.

```python
from census import Census

API_KEY = "API KEY FROM CENSUS BUREAU"

fields = [
    'B01001_001E',  # total pop
    'B01001B_001E',  # Black pop

    'B01001_003E',  # boys
    'B01001_004E',
    'B01001_027E',  # girls
    'B01001_028E',

    'B01001B_003E',  # Black boys
    'B01001B_004E',
    'B01001B_018E',  # Black girls
    'B01001B_019E',

    'B01001_001M',  # total pop
    'B01001B_001M',  # Black pop

    'B01001_003M',  # boys
    'B01001_004M',
    'B01001_027M',  # girls
    'B01001_028M',

    'B01001B_003M',  # Black boys
    'B01001B_004M',
    'B01001B_018M',  # Black girls
    'B01001B_019M',
]


def get_zip_data():
    c = Census(API_KEY)
    # note: default acs5 is most recent (2019 at time of use)
    return c.acs5.get(fields, {'for': 'zip code tabulation area:*', 'in': 'state:42'})


def write_zip_data(data):
    with open('zip_code_pop_2019.csv', 'w') as f:
        import csv
        writer = csv.DictWriter(f, fieldnames=fields + ['state', 'zip code tabulation area'])
        writer.writeheader()
        writer.writerows(data)


if __name__ == '__main__':
    data = get_zip_data()
    print(data)
    write_zip_data(data)
```
