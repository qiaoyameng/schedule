Timezone & Daylight Saving Time
===============================

Timezone in .at()
~~~~~~~~~~~~~~~~~

Schedule supports setting the job execution time in another timezone using the ``.at`` method.

**To work with timezones** `pytz <https://pypi.org/project/pytz/>`_ **must be installed!** Get it:

.. code-block:: bash

    pip install pytz

Timezones are only available in the ``.at`` function, like so:

.. code-block:: python

    # Pass a timezone as a string
    schedule.every().day.at("12:42", "Europe/Amsterdam").do(job)

    # Pass an pytz timezone object
    from pytz import timezone
    schedule.every().friday.at("12:42", timezone("Africa/Lagos")).do(job)

Schedule uses the timezone to calculate the next runtime in local time.
All datetimes inside the library are stored `naive <https://docs.python.org/3/library/datetime.html>`_.
This causes the ``next_run`` and ``last_run`` to always be in Pythons local timezone.

Daylight Saving Time
~~~~~~~~~~~~~~~~~~~~
Scheduling jobs that do not specify a timezone do **not** take clock-changes into account.
Timezone unaware jobs will use naive local times to calculate the next run.
For example, a job that is set to run every 4 hours might execute after 3 realtime hours when DST goes into effect.

But when passing a timezone to ``.at()``, DST **is** taken into account.
The job will run at the specified time, even when the clock changes.

Example clock moves forward:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
When a job is scheduled in the gap that occurs when the clock moves forward, the job is scheduled after the gap.

A job is scheduled ``.at("02:30", "Europe/Berlin")``.
When the clock moves from ``02:00`` to ``03:00``, the job will run once at ``03:30``.
The day after it will return to normal and run at ``02:30``.

A job is scheduled ``.at("01:00", "Europe/London")``.
When the clock moves from ``01:00`` to ``02:00``, the job will run once at ``02:00``.

Example clock moves backwards:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
A job is scheduled ``.at("02:30", "Europe/Berlin")``.
When the clock moves from ``02:00`` to ``03:00``, the job will run once at ``02:30``.
It will run only at the first time the clock hits ``02:30``, but not the second time.
The day after, it will return to normal and run at ``02:30``.

Example scheduling across timezones
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Let's say we are in ``Europe/Berlin`` and local datetime is ``2022 march 20, 10:00:00``.
At this moment daylight saving time is not in effect in Berlin (UTC+1).

We schedule a job to run every day at 10:30:00 in America/New_York.
At this time, daylight saving time is in effect in New York (UTC-4).

.. code-block:: python

    s = every().day.at("10:30", "America/New_York").do(job)

Because of the 5 hour time difference between Berlin and New York the job should effectively run at ``15:30:00``.
So the next run in Berlin time is ``2022 march 20, 15:30:00``:

.. code-block:: python

    print(s.next_run)
    # 2022-03-20 15:30:00

    print(repr(s))
    # Every 1 day at 10:30:00 do job() (last run: [never], next run: 2022-03-20 15:30:00)

Calendar constraints
~~~~~~~~~~~~~~~~~~~~

Fixed daily and weekly jobs can combine allowed weekdays, excluded dates and a
local time window:

.. code-block:: python

    schedule.every().day.at("01:30", "America/New_York") \
        .on_weekdays("monday", "tuesday", "wednesday", "thursday", "friday") \
        .exclude_dates("2026-01-01") \
        .between("22:00", "06:00") \
        .do(job)

``exclude_dates()`` accepts :class:`datetime.date` values or ``YYYY-MM-DD``
strings. Each date is interpreted in the timezone passed to ``at()``; when no
timezone is supplied, the process timezone is used.

If the window end is earlier than its start, the window crosses midnight.
Weekday and excluded-date checks use the final local date in the task
timezone, while the wall-clock time is checked against the window. For
example, a run at 01:30 on Saturday needs Saturday to be allowed and must
not be excluded; its wall-clock time is accepted by ``between("22:00",
"06:00")``. The final local timestamp is checked after DST resolution, so a
spring-forward adjustment cannot move a run back onto an excluded date or
into an unallowed weekday.

When a configured local time does not exist during a spring-forward gap, the
run is moved forward to the valid moment after the gap. If that no longer
satisfies the weekday or window, the scheduler advances by the job's calendar
interval (one day or one week, for example) rather than scanning second by
second. During a fall-back overlap, the first occurrence of the repeated local
time is selected.

Impossible configurations raise ``ScheduleValueError`` while the job is being
defined, for example when a weekly weekday is not allowed, the ``at()`` time is
outside the window, or the interval can never reach an allowed weekday.

Randomized interval jobs created with ``to()`` do not support these calendar
constraints. This includes random second, minute, hour, day and week intervals.
Use a fixed daily or weekly interval when combining ``on_weekdays()``,
``exclude_dates()`` or ``between()``.
