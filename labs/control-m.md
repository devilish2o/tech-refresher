# Lab: Control-M scheduling and recovery

## Goal

Model a business batch flow and reason about dependencies, calendars, reruns and operational recovery. Use a non-production environment or a paper design if no lab instance is available.

## Scenario

A daily inbound file triggers validation, transformation, database load and report generation. The report must not run until the load succeeds.

## Exercises

- Define jobs, folders, calendars and dependencies.
- Distinguish time-based scheduling from event/file-based triggering.
- Model success/failure conditions and a business-day calendar.
- Define rerun behaviour so that duplicate processing is controlled.
- Add SLA expectations, alerts and an operator handover note.
- Design recovery for a missing file, late upstream job, partial load and downstream failure.

## Self-check

Can you explain the difference between rerunning a job and safely replaying a business transaction? What evidence is needed before an operator releases a blocked workflow?

## References

Use the official BMC Control-M documentation for the installed version: [BMC documentation](https://docs.bmc.com/).
