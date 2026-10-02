Calendar constraints
====================

Daily and weekly jobs can be restricted with a combination of three
calendar constraints:

.. code-block:: python

    import schedule

    # Weekday early-morning sync that skips a maintenance calendar
    schedule.every().day.at("02:30", "Europe/Berlin") \
        .on_weekdays("monday", "tuesday", "wednesday", "thursday", "friday") \
        .except_dates("2026-10-13", "2026-10-15") \
        .between("01:00", "04:00") \
        .do(sync_job)

- ``on_weekdays(*days)`` only allows runs on the given weekdays.
- ``except_dates(*dates)`` skips specific local dates, e.g. a
  maintenance calendar. Dates are given as ``datetime.date`` or
  ``YYYY-MM-DD`` strings.
- ``between(start, end)`` only allows runs within a local time-of-day
  window. The window may cross midnight: ``between("22:00", "02:00")``
  allows 22:00-24:00 and 00:00-02:00. Boundaries are inclusive. Without
  an ``at()`` time, the job runs at the start of the window.

Semantics
---------

- Constraints are evaluated in the job's timezone (the ``tz`` argument
  of ``at()``), or in local time when no timezone is configured.
  Excluded dates are interpreted as local dates in that timezone.
- All constraints are re-checked against the *final* resolved local
  moment, so a run that a DST transition moves onto an excluded date or
  outside the window is skipped instead of being scheduled there.
- Daylight saving time edge cases have explicit policies:

  - A local time that does not exist (spring-forward gap) is moved
    forward by the size of the gap, keeping the minutes. For example,
    when 02:30 does not exist because the clock jumps from 02:00 to
    03:00, the job runs at 03:30.
  - A local time that occurs twice (fall-back overlap) resolves to the
    earlier of the two occurrences.

- The next run is found by advancing day-by-day over the local calendar
  (never second-by-second), so scheduling stays fast regardless of the
  constraints.
- Unsatisfiable constraints raise ``ScheduleValueError`` as soon as the
  job is scheduled, e.g. when no allowed weekday remains or the ``at()``
  time lies outside the ``between()`` window.

Compatibility
-------------

Calendar constraints are only supported for:

- daily jobs (``every().day``) and weekly jobs with a fixed weekday
  (e.g. ``every().monday``), always with an interval of 1.

They cannot be combined with:

- second/minute/hour jobs, intervals greater than 1, or weekly jobs
  without a fixed weekday;
- randomized intervals (``every(A).to(B)``). Combining ``to()`` with
  ``on_weekdays()``, ``except_dates()`` or ``between()`` raises
  ``ScheduleValueError`` when the job is scheduled.
