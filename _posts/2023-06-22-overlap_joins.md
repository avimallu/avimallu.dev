---
title: Fast Overlap Joins
permalink: /docked_trucks_in_interval
tags: [python, polars, duckdb, R, data.table, foverlaps, overlap, join]
---

# Premise

I stumbled upon an interesting [Stackoverflow question](https://stackoverflow.com/questions/76488314/polars-count-unique-values-over-a-time-period) that was linked [via an issue](https://github.com/pola-rs/polars/issues/9467) on Polars github repo. The OP asked for a pure Polars solution. At the time of answering the question Polars did not have support for non-equi joins, and any solution using it would be pretty cumbersome.

I'm more of a right-tool-for-the-job person, so I tried to find a better solution.

# Problem Statement

Suppose we have a dataset that captures the arriva and departure times of trucks at a station, along with the truck's ID.

```py
import polars as pl
data = pl.from_repr("""
┌─────────────────────┬─────────────────────┬─────┐
│ arrival_time        ┆ departure_time      ┆ ID  │
│ ---                 ┆ ---                 ┆ --- │
│ datetime[μs]        ┆ datetime[μs]        ┆ str │
╞═════════════════════╪═════════════════════╪═════╡
│ 2023-01-01 06:23:47 ┆ 2023-01-01 06:25:08 ┆ A1  │
│ 2023-01-01 06:26:42 ┆ 2023-01-01 06:28:02 ┆ A1  │
│ 2023-01-01 06:30:20 ┆ 2023-01-01 06:35:01 ┆ A5  │
│ 2023-01-01 06:32:06 ┆ 2023-01-01 06:33:48 ┆ A6  │
│ 2023-01-01 06:33:09 ┆ 2023-01-01 06:36:01 ┆ B3  │
│ 2023-01-01 06:34:08 ┆ 2023-01-01 06:39:49 ┆ C3  │
│ 2023-01-01 06:36:40 ┆ 2023-01-01 06:38:34 ┆ A6  │
│ 2023-01-01 06:37:43 ┆ 2023-01-01 06:40:48 ┆ A5  │
│ 2023-01-01 06:39:48 ┆ 2023-01-01 06:46:10 ┆ A6  │
└─────────────────────┴─────────────────────┴─────┘
""")
```

We want to identify the number of trucks docked at any given time within a threshold of 1 minute *prior* to the arrival time of a truck, and 1 minute *after* the departure of a truck. Equivalently, this means that we need to calculate the number of trucks within a specific window for each row of the data.

# Finding a solution to the problem

## Evaluate for a specific row

Before we find a general solution to this problem, let's consider a specific row:

```py
"""
┌─────────────────────┬─────────────────────┬─────┐
│ arrival_time        ┆ departure_time      ┆ ID  │
│ ---                 ┆ ---                 ┆ --- │
│ datetime[μs]        ┆ datetime[μs]        ┆ str │
╞═════════════════════╪═════════════════════╪═════╡
│ 2023-01-01 06:32:06 ┆ 2023-01-01 06:33:48 ┆ A6  │
└─────────────────────┴─────────────────────┴─────┘
"""
```

This would mean that we need to find the number of trucks that are there between `2023-01-01 06:31:06` (1 minute prior to the `arrival_time` and `2023-01-01 06:34:48` (1 minute post the `departure_time`). Manually going through, we see that `B3`, `C3`, `A6` and `A5` are the truck IDs that qualify.

## Algorithmically deriving the solution

Let's visualize the different situations that we need to consider for the specific row that was discussed above. There are five cases:

![The five different ways a period can overlap.](./assets/001_overlap_joins/overlap_algorithm.png)

Take some time to absorb these cases - it's important for the part where we write the code for the solution. Note that we need to actually tell our algorithm to filter only for Cases 2, 3 and 4, since Cases 1 and 5 will not satisfy our requirements.

## Writing an SQL query based on the algorithm

In theory, we can use any language that has the capability to define rules that meet our algorithmic requirements outlined in the above section to find the solution. However, very few tools have come close to the efficiency and elegance that SQL has in being able to specify the algorithm in an easy to understand manner.

### Introducing the DuckDB package

Again, in theory, any SQL package or language can be used. Far too few however meet the ease-of-use that [DuckDB](https://duckdb.org/) provides:

1. no expensive set-up time (meaning no need for setting up databases, even temporary ones),
2. no dependencies (other than DuckDB itself, just `pip install duckdb`),
3. some very [friendly SQL extensions](https://duckdb.org/2022/05/04/friendlier-sql.html), and
4. ability to work directly on Polars and Pandas DataFrames without conversions

all with [mind-blowing speed](https://duckdblabs.github.io/db-benchmark/) that stands shoulder-to-shoulder with Polars. We'll also use a few advanced SQL concepts, like below:

#### Self-joins

You remember this - a join of a table with itself is a self join. Very few cases where such an operation would make sense, and this happens to be one of them!

#### A bullet train recap of non-equi joins

A key concept that we'll use is the idea of joining on a *range* of values rather than a specific value. That is, instead of the usual `LEFT JOIN ON A.column = B.column`, we can do `LEFT JOIN ON A.column <= B.column` for one row in table `A` to match to multiple rows in `B`. DuckDB has a [blog post](https://duckdb.org/2022/05/27/iejoin.html) that outlines this join in detail, including fast implementation.

#### The concept of `LIST` columns

DuckDB has first class support for `LIST` columns - that is, each row in a `LIST` column can have a varying length (much like a Python `list`), but must have the exact same datatype (like R's `vector`). Using list columns allow us to eschew the use of an additional `GROUP BY` operation on top of a `WHERE` filter or `SELECT DISTINCT` operation, since we can directly perform those on the `LIST` column itself.

#### Date algebra

Dates can be rather difficult to handle well in most tools and languages, with several packages purpose built to make handling them easier - [lubridate](https://lubridate.tidyverse.org/) from the [tidyverse](https://www.tidyverse.org/) is a stellar example. Thankfully, DuckDB provides a similar swiss-knife set of tools to deal with it, including specifying `INTERVAL`s (a special data type that represent a period of time independent of specific time values) to modify `TIMESTAMP` values using addition or subtraction.

### Tell me the query, PLEASE!

Okay - had a lot of background. Let's have at it!

```py
db.query("""
    SELECT
        A.arrival_time
        ,A.departure_time
        ,A.window_open
        ,A.window_close
        -- LIST aggregates the values into a LIST column
        -- and LIST_DISTINCT finds the unique values in it
        ,LIST_DISTINCT(LIST(B.ID)) AS docked_trucks
        -- finally, LIST_UNIQUE calculates the unique number of values in it
        ,LIST_UNIQUE(LIST(B.ID))   AS docked_truck_count

    FROM (
        SELECT
            *
            ,arrival_time   - (INTERVAL 1 MINUTE) AS window_open
            ,departure_time + (INTERVAL 1 MINUTE) AS window_close
        FROM data -- remember we defined data as the Polars DataFrame with our truck station data
    ) A

    LEFT JOIN (
        SELECT
            *
            -- This is the time, in seconds between the arrival and departure of
            -- each truck PER ROW in the original data-frame 
            ,DATEDIFF('seconds', arrival_time, departure_time) AS duration
        FROM data -- this is where we perform a self-join
    ) B

    ON (
    	-- Case 2 in the diagram;
        (B.arrival_time <= A.window_open AND 
        	-- Adding the duration here makes sure that the second interval
        	-- is at least ENDING AFTER the start of the overlap window
        	(B.arrival_time   + TO_SECONDS(B.duration)) >=  A.window_open) OR

        -- Case 3 in the diagram - the simplest of all five cases
        (B.arrival_time >= A.window_open AND 
                                      B.departure_time  <= A.window_close) OR

        -- Case 4 in the digram;
        (B.arrival_time >= A.window_open AND
        	-- Subtracting the duration here makes sure that the second interval
        	-- STARTS BEFORE the end of the overlap window.
        	(B.departure_time - TO_SECONDS(B.duration)) <= A.window_close)
    )
    GROUP BY 1, 2, 3, 4
""")

```

The output of this query is:

```
"""
┌─────────────────────┬─────────────────────┬─────────────────────┬───┬──────────────────┬────────────────────┐
│    arrival_time     │   departure_time    │     window_open     │ … │  docked_trucks   │ docked_truck_count │
│      timestamp      │      timestamp      │      timestamp      │   │    varchar[]     │       uint64       │
├─────────────────────┼─────────────────────┼─────────────────────┼───┼──────────────────┼────────────────────┤
│ 2023-01-01 06:23:47 │ 2023-01-01 06:25:08 │ 2023-01-01 06:22:47 │ … │ [A1]             │                  1 │
│ 2023-01-01 06:26:42 │ 2023-01-01 06:28:02 │ 2023-01-01 06:25:42 │ … │ [A1]             │                  1 │
│ 2023-01-01 06:30:20 │ 2023-01-01 06:35:01 │ 2023-01-01 06:29:20 │ … │ [B3, C3, A6, A5] │                  4 │
│ 2023-01-01 06:32:06 │ 2023-01-01 06:33:48 │ 2023-01-01 06:31:06 │ … │ [B3, C3, A6, A5] │                  4 │
│ 2023-01-01 06:33:09 │ 2023-01-01 06:36:01 │ 2023-01-01 06:32:09 │ … │ [B3, C3, A6, A5] │                  4 │
│ 2023-01-01 06:34:08 │ 2023-01-01 06:39:49 │ 2023-01-01 06:33:08 │ … │ [B3, C3, A6, A5] │                  4 │
│ 2023-01-01 06:36:40 │ 2023-01-01 06:38:34 │ 2023-01-01 06:35:40 │ … │ [A5, A6, C3, B3] │                  4 │
│ 2023-01-01 06:37:43 │ 2023-01-01 06:40:48 │ 2023-01-01 06:36:43 │ … │ [A5, A6, C3]     │                  3 │
│ 2023-01-01 06:39:48 │ 2023-01-01 06:46:10 │ 2023-01-01 06:38:48 │ … │ [A6, A5, C3]     │                  3 │
├─────────────────────┴─────────────────────┴─────────────────────┴───┴──────────────────┴────────────────────┤
│ 9 rows                                                                                  6 columns (5 shown) │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
"""
```

We clearly see the strengths of DuckDB in how succintly we were able to express this operation. However, we also find how DuckDB is able to seamlessly integrate with an existing Pandas or Polars pipeline with zero-conversion costs. In fact, we can convert this back to a Polars or Pandas dataframe by appending the ending bracket with `db.query(...).pl()` and `db.query(...).pd()` respectively. 

# What are the alternatives?

This post is still a work-in-progress - I'll update with additional solutions once we find them!
