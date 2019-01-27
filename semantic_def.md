### Some rebuttals and additions to vezzy-fnord's "Structural and
    Semantic deficiencies in the systemd architecture"

This is my rebuttal and complementary article that corrects some
mistakes in the said article, and complements some of the points they
made with illustrative articles highlighting some core deficiencies in
systemd's implementation.

> Permanent units, though not officially christened as such (the term
  is instead derived from the antonym of “transient unit”, which is
  official terminology) are the standard ones you know which have
  configuration loaded from disk in the form of unit files.

> These have the full breadth of options corresponding to their type,
  and are subject to the unit file installation logic for managing
  their configuration in the file system ([Install] directives).

There are very strong fundamental reasons why [Install] exists, and it
has very little to do with "a way to enable units" more with a
consequence of a design decision: lazy loading of units.

Suppose that systemd allowed you to declare RequiredBy= in the unit
itself, and you will think that everything should get pulled in
easily. The difference here is that at boot, systemd follows forward
dependencies from the default target. If the target does not specify
any dependencies, then the internal transaction builds will not be
able to add jobs for units further, and systemd only loads units that
are actually referenced from these forward dependencies (to minimize
the cost of having to load all units). That means that it sees no
Requires= as an inverse of RequiredBy=target in all the other units,
and nothing will start up. This is why [Install] is used to establish
filesystem symbolic links that translate to the forward dependency for
the target, and then systemd can build the initial transaction out of
that.

> Transient (executable) units ...

Agreed, only stuff related to cgroups is easily modifiable. Currently,
set-property is broken wrt transient units that cannot be created on
disk (scopes in particular), and even not specifying --runtime will
make a drop-in that goes away on a reboot (given the reasoning that
transient objects should not leave around persistent state). Open bug
here: https://github.com/systemd/systemd/issues/7106

> Transient (non-executable) units

Snapshots are gone, but that hasn't rid systemd of the weirdness and
corner cases associated with every unit type. Device units misuse jobs
to wait for the appearance of a device, as there is no payload to
execute except this. Similarly, a stop job can do nothing but wait for
the device to go away. These can then be scheduled by the manager in
case something triggers a start job for it to correctly use the device
as a dependency.

This however manifests into a semantics issue. Jobs do not encapsulate
a unit specific action for every unit type. Compare the start job of a
service, where the state machine keeps it in waiting until the
previous state transition is over. This means that if a unit is
stopping on a stop job request, and one uses the replace job mode
(default) and starts it, a start job replaces the previous stop job,
but keeps waiting for deactivation to finish.

This is in stark contrast to job replacement in device units, where
replacing the start job would effectively also abort the waiting for
the appearance of device, and instead switch that to waiting for the
disappearnace of it.

This is also why systemctl cancel <JOB_ID> of a start job of a service
still proceeds with the activation of the unit, as that is in no way
encapsulated by the job.

See: https://github.com/systemd/systemd/issues/11452
See: https://github.com/systemd/systemd/issues/11526#issuecomment-457592737
See: https://github.com/systemd/systemd/issues/11521#issuecomment-456308500
See: https://github.com/systemd/systemd/issues/11499

> Permanent task-based units unassociated with jobs

All of them enqueue jobs, and the state machine is responsible for, on
performing the action associated with the job type, on success,
transition to a state so that the job's intended state and the target
unit state match and job can be finished.

> Job queuing

Jobs aren't really unit types internally (they were kind of related to
the Meta unit type in the early days must be what is confusing the
author). They are just state change requests for a unit. They can be
thus scheduled independently of one another, wait and block on one
another through ordering dependencies (Before= and After= are solely
properties of a job anyway). They also have their own GC logic that
kicks in when no bus client is responsible for it, and nothing is
ordered against it. They hence undergo deletion in and get freed.

One important property of jobs in systemd is that there can be only
one per unit. This has multiple implications, some of which are
obvious, and some of which are not.

For instance, of all the job types, JOB_RESTART is a little
special. This first performs the action on the unit equivalent to that
of JOB_STOP (unit_stop), but then when the job completes with the
JOB_DONE result (i.e. success), it will instead convert itself to
JOB_START. While not obvious immediately, this is a workaround for the
lack of being able to queue multiple jobs on a single unit. To also
implement waiting on a unit's transition state, the unit itself may
decide if a job can be made runnable just yet. This means that if you
construct the following scenario:

stop A -> installs stop job -> runs stop job
start A -> replaces stop job and installs start job

The start job is now going to be put in the JOB_WAITING state as long
as unit is deactivating, because unit_start returned -EAGAIN. Then,
when the next sub state change happens, unit_notify function will add
this job to run_queue, where manager will then try to dispatch it and
see if it is runnable now. This process repeats until it happens.

Compare this to devices, where replacing a job then also means not
waiting for the invoked operation to finish. This is a major
discrepancy in the internal model, and the higher level overview
presented to the user.

Job modes are while considered a job property, actually end
influencing both the job and the transaction it ends up
generating. More on that later however.

Merging too happens inside a transaction, and across transaction (on a
running unit's installed job). While it may appear neat at first, it
is in some cases used to again work around the lack of being able to
queue multiple jobs on a unit. We will note this later.

> The transaction manager

The transaction builder is broken in many ways. It will produce
incorrect results, and often suffers from the lack of being able to
compute the active unit state, which has some interesting effects. We
will see how, through real world examples.

It is true that when you trigger a job for a unit, the first thing the
manager does is (in the most common case, let's say start job), add a
job for your unit when building a transaction, and then recursively
add it for all your dependencies. This job being triggered is the
anchor job, and the job mode used will influence the entire
transaction (and it's result upon activation).

Let's consider a simple case of a target being started manually, it
gained Wants= + After= on multiple units through the .wants/ symlinks,
and is now being started by the user. The first thing to do is adding
a start job for this target unit, and then setting it as the anchor
job for our transaction (to be computed). Then, systemd will walk the
UNIT_WANTS property for it iterating over every unit and triggering a
start job for each one of those, in a recursive manner (that is,
repeat this iteration for its dependencies). Roughly, when this
process is over, the transaction has an entire set of jobs that are
now ready for consistency and mergeability checks, and then activation
(and installation) on all their units.

Let's pause and ponder over a few details:

1) systemd never considers if the said unit is active or not, i.e.,
the state of the unit being recursively iterated over for dependency
triggers.

	This is actually a major point, and has lead to bugs in the
	systemd implementation. Let's look at the first one:

	https://github.com/systemd/systemd/issues/11338

	This bug report illustrates a test case where the unit Wants=
	a unit that Requires= a non-existent unit.

	Despite that, the second unit starts up successfully. This may
	seem an uncommon case to hit, but is effortless with a service
	enabled at boot (because targets use Wants=). So, as of this
	writing, many of your services might be starting even if you
	have a strong dependency on a non-existent unit.

	The fix may seem simple, that is, when propagating the return
	value from lower levels of recursion, also delete the job that
	couldn't have been queued otherwise, not leaving stray jobs
	around. It isn't. It breaks when you have two units in
	Requires= of the second unit, where the first one exists and
	the second one does not. On recursion, depending on the order
	of iteration, the existing one may get a job enqueued, while
	its dependent may have it deleted due to the other one not
	existing, or none of the three get enqueued. The fix would
	hence produce undeterministic results.

	One other approach pointed out here:
	https://github.com/systemd/systemd/issues/4937#issuecomment-376170343
	is to do a depth-first traversal, working one's way from the
	bottom up. Sounds fine until you realise the problem desribed
	above will just reverse itself. You will still have stray jobs
	depending on which of the two were worked on first.

	Maybe making this atomic would work? Commit only after all
	constraints are satisified.

	Anyway, the one above might be something many feel is a bug
	and not a design issue. Let's try to make our case a little
	stronger and consider:

	https://github.com/systemd/systemd/issues/11440

	The bug describes the entire scenario well, and there are two
	key problems: systemd computes and propagates jobs on units
	that already need no state transition (i.e. stop on something
	already stopped, and so on), and systemd is not enable to
	resolve conflicting job types inside the same transaction on
	the same unit (unlike conflicting jobs across transactions
	where they can replace the installed one).

	The first bit is a documented detail now, but hints towards a
	much bigger problem. The sole reason systemd does this
	(i.e. propagate stop jobs on a unit already stopped) is that
	it cannot know if a unit is already stopped completely and
	will not change state between time of check and time of
	activation. There are many reasons why, and not having an
	installed still doesn't mean such a thing cannot happen. For
	example, it may just be waiting for the restart timer to
	expire (systemd *can* know this, but the list of things to
	check will just keep on growing). It may also be the case that
	a restart job was in progress (which is why it uses the
	auto-restart sub state in that case).

	All in all, there are many caveats to not triggering
	propagation of jobs when the state change will be a no-op. In
	some cases, it may not make sense, and in some, it may, and it
	is not easy to figure out when to do what. This also manifests
	in other problems.

	Consider a oneshot that another unit Requires=. It does that
	for the sole purpose of failing in case the oneshot
	fails. However, Requires= also ensures that stop requests on
	oneshot stops the required unit. Having no distinction between
	dependency and explicit jobs can mean if you have another unit
	that Conflicts=oneshot.service, and the oneshot is already
	dead, starting this unit will take down the service you
	have...

	Things like these are hard to reason about in such automatism,
	and these are all very core implemenation details leaking out.

	The other big issue was not being able to resolve conflicting
	job types in the same transaction. This needs some overview on
	what merging and collapsing is inside and outside a
	transaction.

	Merging serves two functions, merge compatible jobs into a
	satisfying job type that makes the unit reach the intended
	state. It can happen inside a transaction being computed, or
	on a job already installed on a unit (with some things to keep
	in mind to not have adverse effects). The other function is
	working around the lack of queue of jobs on a unit. Collapsing
	if of little interest here as this is not a stateful job type.

	So, we encounter a case of restart and stop being triggered
	for the same unit in a transaction. Merging cannot satisfy
	this, as neither restart can upgrade to stop, nor stop to
	restart. Also, we cannot "replace" jobs as something is yet to
	be installed on a unit anyway (and we replace whatever is
	installed on unit with what we produce in this transaction).

	systemd completely decides to bail in this case! Also, since
	there is no way to queue multiple jobs on a unit, stop and
	restart cannot be both enqueued together on the unit; also,
	depending on order of iteration it may be that restart is
	added first and stop later, in which case even if there was a
	queue, the result would be different on execution.

	Jobs however can replace one another across transactions, and
	the funny thing about this is that if it happens while a state
	transition is going on, the installed job is compared on every
	state change to what the reached state is, say a restart will
	end up stopping the unit if one triggers a stop job and
	replaces it while it is running. This results in multiple
	issues that have manifested in real life:

	See:
	https://github.com/systemd/systemd/issues/11456#issuecomment-455760432

	Here, a unit, due to a race created through use of
	ExecStopPost=, is held in the deactivating stage while the
	process that was running went away, enough for a propagated
	job to come in and install itself, attributing the state
	change to itself (as the way systemd would detect it was an
	abrupt stop was the state change to happen without an
	installed job's involvement). stop gets EALREADY from the
	state machine so it just waits for completion, and the service
	will not be restarted.

	This was worked around for a very specific case when
	StopWhenUnneeded=true is involved,
	https://github.com/systemd/systemd/issues/1154#issuecomment-455835515
	but did little to help problems stemming from the root cause.


> Ordering cycles

Mostly correct, and also add that if you use strong dependencies
amongst cyclic services, it fundamentally becomes unbreakable for
systemd, as the matters_to_anchor bool will be true for all such jobs,
which will cause systemd to bail on breaking the cycle.

> Destructive transactions

These not only happen when JOB_NOP tries to merge into anything but
JOB_NOP (which is actually not very likely to happen due to external
requests), but also when the said job is irreversible in nature (so
nothing but an explicit cancel can remove it from the unit's slot), or
the transaction uses a job mode like JOB_FAIL that instead fails the
transaction if resolution involves replacing jobs/cancelling them.

This manifests into two major issues, one of which has been fixed, by
now using the JOB_REPLACE job mode, however the other being a little
harder to fix due to the job action encapsulation inconsistency,
making it unsafe to readily replace running jobs on units:

See: https://github.com/systemd/systemd/issues/11305
See: https://github.com/systemd/systemd/issues/11526
See: https://github.com/systemd/systemd/issues/3260

...
