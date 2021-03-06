Prerequisite Tasks and Subtasks
===============================
<h2>
Xylar Asay-Davis <br>
date: 2017/06/12 <br>
</h2>

<h3> Summary </h3>

Currently, no tasks depend on other tasks to run.  However, in order to allow
multiple plots to be generated simulataneously, it is desirable to break tasks
into multiple subtasks, and some of these subtasks will need rely on data from
other subtasks.  It is also conceivable that multiple tasks could rely on the
same data (e.g. a common climatology dataset). The proposed solution to this
problem is to allow "prerequisite tasks" to a given analysis task.  The task
will only run after the prerequisite task(s) have completed.  Prerequisite
tasks could be used to build up a sequence of analysis tasks in several steps.
Some of these steps could be shared between analysis tasks (e.g. computing
single data set and then plotting it in various ways).  Implementation of this
design will be considered a success if dependent tasks only run once their
prerequisite tasks have completed successfully.

<h2> Requirements </h2>

<h3> Requirement: Define prerequisite tasks <br>
Date last modified: 2017/06/12 <br>
Contributors: Xylar Asay-Davis
</h3>

A simple mechanism (such as a list of task names) exists to define prerequisite
tasks of each analysis task.

<h3> Requirement: Add prerequisites to task list <br>
Date last modified: 2017/06/12 <br>
Contributors: Xylar Asay-Davis
</h3>

Given a task that we want to run, a mechanism must exist for adding its
prerequisites (if any) to the list of tasks to be run.

<h3> Requirement: Holding dependent tasks <br>
Date last modified: 2017/06/12 <br>
Contributors: Xylar Asay-Davis
</h3>

Dependent tasks (those with prerequisites) must be prevented from running until
their prerequisites have successfully finished.

<h3> Requirement: Cancel dependents of failed prerequisites <br>
Date last modified: 2017/06/12 <br>
Contributors: Xylar Asay-Davis
</h3>

If a prerequisite of a dependent tasks has failed, the dependent task should
not be run.

<h2> Algorithmic Formulations </h2>

<h3> Design solution: Define prerequisite tasks <br>
Date last modified: 2017/09/19 <br>
Contributors: Xylar Asay-Davis
</h3>

Each task will be constructed with a list of the names of prerequisite tasks.
If a task has no prerequisites (the default), the list is empty.

<h3> Design solution: Add prerequisites to task list <br>
Date last modified: 2017/10/11 <br>
Contributors: Xylar Asay-Davis
</h3>

A recursive function will be used to add a given task (assuming its
`check_generate` method returns `True`, meaning that task should be generated)
and its dependencies to a list of analyses to run.  The code (with a few
error messages removed for brevity) is as follows:
```python
analysesToGenerate = []
# check which analysis we actually want to generate and only keep those
for analysisTask in analyses:
    # update the dictionary with this task and perhaps its subtasks
    add_task_and_subtasks(analysisTask, analysesToGenerate)

def add_task_and_subtasks(analysisTask, analysesToGenerate,
                          callCheckGenerate=True):

    if analysisTask in analysesToGenerate:
        return

    if callCheckGenerate and not analysisTask.check_generate():
        # we don't need to add this task -- it wasn't requested
        return

    # first, we should try to add the prerequisites of this task and its
    # subtasks (if they aren't also subtasks for this task)
    prereqs = analysisTask.runAfterTasks
    for subtask in analysisTask.subtasks:
        for prereq in subtask.runAfterTasks:
            if prereq not in analysisTask.subtasks:
                prereqs.extend(subtask.runAfterTasks)

    for prereq in prereqs:
        add_task_and_subtasks(prereq, analysesToGenerate,
                              callCheckGenerate=False)
        if prereq._setupStatus != 'success':
            # this task should also not run
            analysisTask._setupStatus = 'fail'
            return

    # make sure all prereqs have been set up successfully before trying to
    # set up this task -- this task's setup may depend on setup in the prereqs
    try:
        analysisTask.setup_and_check()
    except (Exception, BaseException):
        analysisTask._setupStatus = 'fail'
        return

    # next, we should try to add the subtasks.  This is done after the current
    # analysis task has been set up in case subtasks depend on information
    # from the parent task
    for subtask in analysisTask.subtasks:
        add_task_and_subtasks(subtask, analysesToGenerate,
                              callCheckGenerate=False)
        if subtask._setupStatus != 'success':
            analysisTask._setupStatus = 'fail'
            return

    analysesToGenerate.append(analysisTask)
    analysisTask._setupStatus = 'success'
```

<h3> Design solution: Holding dependent tasks <br>
Date last modified: 2017/10/11 <br>
Contributors: Xylar Asay-Davis
</h3>

Each task is given a `_runStatus` attribute, which is a `multiprocessing.Value`
object that can be shared and changed across processes.  A set of constant
possible values for this attribute, `READY`, `BLOCKED`, `RUNNING`, `SUCCES` and
`FAIL` are defined in`AnalysisTask`.  If a task has no prerequisites, initially
`_runStatus = READY`; otherwise `_runStatus = BLOCKED`.  Any `READY`
task can be run (`_runStatus = 'running'`).  Any task that finishes is given
`_runStatus = SUCCESS` or `_runStatus = FAIL` (I know, not grammatically
consistent but compact...).

When a new parallel slot becomes available, all `BLOCKED` tasks are checked
to see if any prerequisites have failed (in which case the task also fails) or
if all prerequisites have succeeded, in which case the task is now `READY`.
After that, the next `READY` task is run.

<h3> Design solution: Cancel dependents of failed prerequisites <br>
Date last modified: 2017/06/12 <br>
Contributors: Xylar Asay-Davis
</h3>

Same as above: When a new parallel slot becomes available, all `BLOCKED`
tasks are checked to see if any prerequisites have failed (in which case the
task also fails).

<h2> Design and Implementation </h2>

The design has been implemented in the branch
[xylar/add_mpas_climatology_task](https://github.com/xylar/MPAS-Analysis/tree/add_mpas_climatology_task)

<h3> Implementation: Define prerequisite tasks <br>
Date last modified: 2017/10/11 <br>
Contributors: Xylar Asay-Davis
</h3>

`AnalysisTask` now has an attribute `runAfterTasks`, which default to empty.
Prerequisite tasks can be added by calling `run_after(self, task)` with the
task that this task should follow.

<h3> Implementation: Add prerequisites to task list <br>
Date last modified: 2017/10/11 <br>
Contributors: Xylar Asay-Davis
</h3>

`build_analysis_list` in `run_mpas_analysis` has been modified to call a
recursive function `add_task_and_subtasks` that adds a task, its prerequisites
(if they have not already been added) and its subtasks to the list of tasks
to run.

<h3> Implementation: Holding dependent tasks <br>
Date last modified: 2017/06/12 <br>
Contributors: Xylar Asay-Davis
</h3>

The `run_analysis` function in `run_mpas_analysis` has been updated to be aware
of the status of each task, as described in the algorithms section.

<h3> Implementation: Cancel dependents of failed prerequisites <br>
Date last modified: 2017/06/12 <br>
Contributors: Xylar Asay-Davis
</h3>

Again, the `run_analysis` function in `run_mpas_analysis` has been updated to
be aware of the status of each task, as described in the algorithms section.

<h2> Testing and Validation </h2>
<h3> Date last modified: 2017/06/12 <br>
Contributors: Xylar Asay-Davis
</h3>
All plots will be tested to ensure they are bit-for-bit identical to
those produced by `develop` for all tests defined in the `configs/edison`
and `configs/lanl` directories.  Task will be run in parallel and I will
verify that no dependent tasks run before prerequisite tasks have completed.
