Thread-safe scheduling
######################

The default scheduler keeps its historical behavior. When more than one
thread may call ``run_pending()``, ``run_all()`` or ``Job.run()`` for the
same scheduler, use a thread-safe scheduler:

.. code-block:: python

    import schedule

    scheduler = schedule.Scheduler(thread_safe=True)

    # This is the equivalent shorthand:
    scheduler = schedule.ThreadSafeScheduler()

A due job is selected and marked as running in one lock-protected operation.
The scheduler lock is released while the job function executes, so a job can
add jobs, cancel itself or another job, and change its own schedule from
inside the callback. Changes to ``next_run`` are preserved when the job
finishes.

If two scheduler entries encounter the same due job at the same time, only
the first one runs it; the other skips it. If the job function raises an
exception, its running marker is released so the same job can be picked up
again later.

