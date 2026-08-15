# FIFA World Cup 2026 Schedule Optimization

This project is a three-stage mixed-integer optimization pipeline built with Gurobi to redesign the group-stage schedule of the 2026 FIFA World Cup, based on the Canadian Operational Research Society (CORS) OR Challenge 2026.

## Background

The 2026 tournament expands to 48 national teams playing across 16 host cities in Canada, Mexico, and the United States, making it the largest FIFA World Cup to date. That scale creates a genuinely hard scheduling problem. Teams need fair group assignments, travel between host cities has to stay reasonable, rest between matches needs to be adequate, and kickoff conditions (heat, humidity, rain) vary a lot by city and time slot.

No cleaned dataset, objective function, or constraint list was provided as part of the challenge. This involved collecting and cleaning the underlying data (team rankings, host city/stadium details, distances, time zones, and historical weather), defining metrics to evaluate schedule quality, and building optimization models to propose an improved schedule.

## Approach

The problem is broken into three sequential Gurobi models, each one feeding its output into the next:

| Stage | Model | Decides | Depends on |
|---|---|---|---|
| 1 | Group Assignment | Which 4 teams go in each of the 12 groups | Team rankings only |
| 2 | Venue & Day Scheduling | Which stadium and which day each match is played | Model 1 output |
| 3 | Kickoff Date & Time | Exact date and hour for each match | Model 2 output |

Models 1 and 2 are framed as **non-preemptive goal programming** problems. Rather than hard-constraining values like rest days or timezone shift to an exact number, target values are set (e.g. 3 days of minimum rest, 1 hour of maximum timezone shift) and deviation variables (`_plus` / `_minus`) absorb the gap above or below that target. Those deviations are then penalized in a single weighted objective function instead of being optimized in strict priority order.

### Model 1: Group Assignment

Assigns all 48 teams into 12 groups of 4, subject to:
- Each of the three host nations (Canada, Mexico, USA) is pre-seeded into its own group
- Confederation limits per group (max 2 UEFA teams, max 1 from any other confederation)
- A target "point range" per group, derived from the gap between the average FIFA ranking points of the top 12 teams and the bottom 12

The objective minimizes two things simultaneously: the spread between the strongest and weakest group averages (weighted 15x), and how far each group's internal point range deviates from the target range (weighted 1x). Solved with a 15-minute time limit and a 1% MIP gap tolerance.

### Model 2: Venue & Day Scheduling (Travel Burden)

Assigns each of the 72 group-stage matches to one of the 16 venues and one of 17 tournament days, minimizing total team travel distance while keeping rest and jet lag in check. Key constraints:

- Round 2 matches for a group must occur after all Round 1 matches for that group, and Round 3 must occur after Round 2
- Each stadium hosts at most one match per day
- Stadium usage is capped proportionally to seating capacity, so matches don't all funnel into the two or three largest venues
- A team can't play its two consecutive group matches at the same venue

Goal programming targets: 3 days minimum rest between a team's matches (penalty weight 50) and a maximum 1-hour timezone shift between consecutive venues (penalty weight 100). A small negative weight on total stadium capacity in the objective nudges the solver toward larger stadiums when travel/rest/timezone costs are otherwise similar. Solved with a 30-minute time limit.

### Model 3: Kickoff Date & Time (Weather)

Given the venues and days from Model 2, assigns an exact date and kickoff hour to each match to minimize exposure to poor playing conditions, subject to the rest days computed in Model 2 and a limit of 8 matches per day. A weather penalty score is computed per city/date/hour combination from temperature, humidity, and precipitation:

- +2 for temperature above 28°C or below 10°C, +1 for 22–28°C or 10–15°C
- +2 for humidity above 60%
- +2 for rain above 5mm, +1 for any rain up to 5mm
- Score forced to 0 for climate-controlled/retractable-roof stadiums

## Repository Contents

- `final_model.ipynb`: Full pipeline, including data loading, current-schedule analysis, and all three Gurobi models
- `cleaned_data.xlsx`: Cleaned input data (team rankings, host cities/stadiums, distance matrix, current published schedule, time zones, historical weather by city, and one example solved solution)

## Running the Models

### Setup

```bash
git clone https://github.com/nitya-balaji/fifa-schedule-optimization.git
cd fifa-schedule-optimization
pip install pandas gurobipy matplotlib openpyxl
```

Update the file path in the first code cell of `final_model.ipynb` to point to `cleaned_data.xlsx`. Solving the models requires a Gurobi license, free for academic use through Gurobi's academic program.

### Usage

Run the following cells in order, since each model depends on the output of the one before it:

1. Run the data-loading and helper-function cell to populate all sets and parameters.
2. Run the current-schedule analysis cells to establish a baseline (travel distance and timezone shift under the currently published schedule).
3. Run Model 1. This produces `team_groups.csv`, which Model 2 depends on.
4. Run Model 2 (data cleaning and the model itself). This produces `schedule_model2.csv` and `team_summary_model2.csv`, which Model 3 depends on.
5. Run Model 3. This produces the final schedule as `model3_schedule_output.csv`.
6. Run the final weather-comparison cells to compare current vs. optimized schedules across both models.

**Note:** Models 1 and 2 do not have to converge to a provably optimal solution within their time limits (both fall back to the best feasible solution found rather than requiring `OPTIMAL` status, so results can vary slightly between runs).

## Tools

Python, pandas, Gurobi (gurobipy), matplotlib