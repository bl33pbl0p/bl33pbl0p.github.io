# systemd - Job propagation through dependencies

systemd's dependencies trigger propagation of jobs in transaction, to put it simply. Ordering is exclusive (which is a useful property).

Ordering is recommend as other wise failure of a job cannot have an effect on the other job (that technically is waiting but without ordering in reality they both run in parallel). This is because jobs work by cancelling the waiting one with a job result (JOB_DEPENDENCY for instance when a job is pulled in for UNIT_REQUIRES). Once the job completes, it cannot be cancelled, hence the unit activation state cannot be altered (and should not be, ofcourse). However, this property can be exploited to *only* react to state change events in a service and not bind oneself to the result of that job (BindsTo= with Before= is a perfect example of this).

This is a simplified explanation of systemd's dependency types, viewed as propagation through traversal during transaction building. Ordering types serve as a mechanism to order jobs in a transaction, otherwise jobs execute in parallel. I am also strictly against making them non-orthogonal, as there is already quite a lot of stuff to keep in mind, I'd like to stick to an explanation where one can reasonably reason about the transactional dependency engine of PID 1.

This is a picture of dependencies like Wants=, Requires=, BindsTo=, PartOf=, etc as purely propagative relationships, and the dependency like semantics of them when used in combination with ordering (in fact, this is close to how it works internally, hence a unifying model to explain things).

It is key to keep in mind two terms, used throughout this document, explicit jobs and implicit jobs.

* Explicit jobs: Those triggered by the service manager, either through the bus, as part of transaction dependency addition, or other ways jobs get pulled by transactions the manager builds. Wants=, Requires=, BindsTo=, PartOf= work under these job types, and cause installation of other jobs in the transaction. Ordering is completely orthogonal, and we stress it again.

* Implicit jobs: Those triggered by state changes of a unit, when the process crashes, is killed, or exits cleanly, or the unit itself goes to some state on its own if not encapsulating a process. Usually, state is altered only through jobs for a unit in systemd, hence these are special in that they trigger retroactively start/stop actions which consititue a transaction of their own. The unit state machine has sub states that map, through a translation table, to generic unit states like active, inactive, failed, activating, deactivating, and reloading. We will discover through this reasoning, for instance, why BindsTo= is made heavy use of in conjuction with device units, though this is not a requirement or a design decision (however the usecase is certainly an influence). BindsTo= is also the only dependency type triggering on implicit state changes in units.

One would usually associate state changes to dependencies, but we will associate jobs triggered on the subject unit with how that propagates, based on these dependency relationships, to object units. State changes mapping to dependencies have credence, but are hard to reason about. Saying that device units go straight from active -> inactive on devices being unplugged is why PartOf= won't trigger is perhaps fine, but saying that there is no stop job triggered is why things that the device internally has in ConsistsOf= won't get those stop jobs propagated is another way of making clear why it actually doesn't work. The fact that BindsTo= then triggeres a stop job when the subject transition to the inactive state is why BindsTo= is made heavy use of with device units.

Similarly, when does a unit with Requires= on others stop? Explicit requests? Well, anything that triggers a JOB_STOP on that object unit will cause it to walk down RequiredBy= and trigger one JOB_STOP for each of these, installing and enqueuing it in the transaction for execution, based on the order defined by After=/Before=. So, a PartOf= propagation on the object will trigger a JOB_STOP on it, triggering as a consequence a JOB_STOP on the subject i.e. us.

Summary for property used for walking the propagation on explicit job triggers(manager triggered jobs):

| Relationships/Jobs |   JOB_START for all in   |JOB_STOP for all in | JOB_RESTART (as try-restart) for all in|JOB_RELOAD for all in| 
|--------------------|--------------------------|--------------------|----------------------------------|------------------|
| Wants=/Requires=   |     Wants=/Requires=     |  NULL/RequiredBy=  |       NULL/RequiredBy=           |    NULL/NULL     |
| PartOf=            | NULL (so no dep pulling) |    ConsistsOf=     |         ConsistsOf=              |       NULL       |
| BindsTo=           |        BindsTo=          |     BoundBy=       |          BoundBy=                |       NULL       |
| ReloadPropagators= |          NULL            |       NULL         |            NULL                  | Opposite of used |

Summary for propagation on implicit job triggering (unit state changes without systemd's involvement): (jobs on unit state change by process only triggers Restart=, so only JOB_RESTART counts, which acts like JOB_STOP before conversion to JOB_START after result=done). Those that don't trigger restart job may hit failed/inactive state directly without involving stop jobs that transition them through deactivation stage (device units), in which case BindsTo= triggers a JOB_STOP on the object unit. This is referred to in systemd circles as abrupt disappearance of units, but is in essence just a direct state transition (devices have only three states anyway). Hence, BindsTo=/BindTo= (same thing) is established as way to react to all state changes in a unit, implicit or explicit and queue the relevant job on the object.

|        Relationships           |         inactive->active/activating      |   active->inactive/failed   |
|      /State Transition,        |       Eg. (auto-restart)->(start) or     | Eg.(failed)->(auto-restart) |
|        Example(s) and          |              (start)->(running)          |    or \*->inactive/failed   | 
|        JOB triggered           |  JOB_START (converstion of JOB_RESTART)  | JOB_STOP ( == JOB_RESTART ) |
|--------------------------------|------------------------------------------|-----------------------------| 
|       Wants=/Requires=         |                 NULL/NULL                |          NULL/NULL          |
|            PartOf=             |          NULL (so no dep pulling)        |             NULL            |
|            BindTo=             |                  BoundBy=                |           BoundBy=          |

On state change, manager matches against the new state and triggers jobs for its dependencies (walking through these properties) that constitute a transaction.

Now that the distinction has been made clear, one might ask: What if BindsTo= causes a job of type JOB_STOP to be queued, will Restart= trigger? The answer is no, as the manager queues an explicit job (this was filed as a bug for systemd, the reader might be able to know why exactly things work this way).

## Other notes
Ordering is orthogonal, and influences the order in which jobs are queued. Hence, BindsTo= and Before= produces a meaningful result of only reacting to ALL state changes of the bound unit but not neccessarily following the condition of it being explicitly active. Therefore explaining it as "this will make unit stay active as long is unit BoundTo= is active" is a little incorrect, since using Before= means our job starts up and completes before theirs, and stops and completes after theirs (but due to propagation effects of BindsTo=, it can trigger JOB_STOP type jobs for us when that unit we're bound to enters the inactive/failed state or gets a stop job queued for itself).

* Requires=

When Subject
    
    has a JOB_START enqueued

Object  will
   
    have a JOB_START job enqueud
    
When used without After=/Before=, if Object's JOB_START fails and Subject's JOB_START hasn't finished running (both were already runnable, and were run concurrently, the difference being atleast one has to be dispatched before another in theory, but there is no waiting), then Subject's JOB_START will be cancelled with the JOB_DEPENDENCY job result.

When used with After=, if Object's JOB_START fails, then Subject's JOB_START will be cancelled with the JOB_DEPENDENCY job result, however, the Subject's job can complete before the Object's job fails, therefore invalidation won't work (this is why Requires= doesn't stop already started units). Understanding this interaction in terms of jobs gives a clear idea of what exactly happens.

When used with Before=, no job cancelation will ever happen, hence during startup Requires= + Before= can be used as Wants= but explicit stop jobs on the object will propagate those to us (and also cause us to stop after them). This behaves like PartOf= for propagation effect but does not trigger JOB_RESTART jobs on us.


When Object
    
    Has a JOB_STOP enqueued by systemd

Subject will

    have a JOB_STOP enqueued (because the object's job walks down RequiredBy= to trigger propagation as part of the transaction (which also ensures symmetric ordering ofcourse).


When Object

    Has a JOB_RESTART enqueued by systemd

    Has a JOB_RESTART enqueued because process died and Restart= queued such jobs if configured (unit state change internally)

Subject will
    
    have a JOB_TRY_RESTART propagated if it was systemd enqueueing the JOB_RESTART

Requires= alone without After= will pull in start jobs as part of the transaction for both the subject and the object, but the failure mode of the subject's job depends on whether the object's job fails before it starts up (and our job becomes uncancellable so its failure has no effect anymore). Using After= ensures we never start up in case the object in question fails to start up. One can also use Before=, but that has the effect of starting us before the object, only pulling them in, but their failure not causing us to stop (as our job already completes before them). So, After= and Before= just serve as means of ordering oneself in a transaction. It is recommended often to use Wants= especially in targets unless Requires= is needed as that can usually cause ungraceful failures in the transaction, and if a target unit uses Requires=, and there exist cycles among units that use Requires= for same strong dependency effects, the cycle becomes unbreakable and the boot
stalls (as matter_to_anchor boolean is true).

The Condition thing mentioned in the man page outlines that failing conditions are usually treated as JOB_DONE internally, so failing conditions would not cause a transaction failure.

* Requisite=

TBD

Requisite= MUST be used with After=, otherwise it is undeterministic due to the nature of its implementation. In particular, Requisite= causes the manager to trigger a job of type JOB_VERIFY_ACTIVE, however no After= will, unpredictably dispatch one of the other jobs first (our start job or the verification job), and our job may or may not complete by then, and hence, may or may not be cancellable. If we complete by then, we won't be shut down, and Requisite= will have no effect. This directive serves the same function as that of a systemctl is-active foobar.service, with a positive outcome as requirement for success.

An example of Requisite= would be a unit that needs time synchronisation to work. time-sync.target is started by a ntp daemon once synchronisation is done. Our unit must be started iff time-sync.target is started, but it should not start time-sync.target itself, since that must be done at the right time by the ntp daemon.

* Wants=

When subject

    has a JOB_START enqueued

Object will
    
    have a JOB_START enqueued

Nothing happens on any jobs queued on object, no propagation.

This is the recommended way and provides a graceful fallback path when object units fail to start up, and is very useful for target units (see why everyone uses WantedBy=). Weaker dependencies also mean systemd can easily break cycles when it detects one, by deleting those that do not matter to the anchor job (the one requested), and some other means in place.

Having After=/Before= doesn't make much of a difference in these wrt failure modes of jobs (as they never cancel others really). Units queued as part of UNIT_WANTS come with job mode fail, which cancels them on finding conflicting jobs in queue.

* BindsTo=

Triggers for JOB_STOP/JOB_START/JOB_RESTART, and service state transition that are local/implicit (but in order, i.e. for restart, JOB_START after conversion waits if it ordered using After=, so that the condition of the subject being active only when object is is strictly followed). Eg; B BindsTo=A, A is restarted, then flow happens like this:
    A: JOB_RESTART, result=done, converted to JOB_START, enqueued and waiting on B's JOB_RESTART
    B: JOB_RESTART, result=done, converted to JOB_START, enqueued and waiting on A's JOB_START
    A: JOB_START, result=done, B: JOB_START, result:done
    
If one uses BindsTo= and Before=, naturally, the positive job will be ordered Before= that of A, and any state change in A will propagate those jobs to B, but the ordering of B will encapsulate A under it. BindsTo= hence of itself has NO ordering at all, and just serves to exclusively track explicit state changes causes by jobs (installed by systemd) or implicit (by unit itself) state change and react to it. This can hence be used to create a job that is free of invalidation/failing from ordered jobs in the transaction yet reacts to *ALL* state changes of a unit.

also,

When object
    
    enters failed state
    enters inactive state

Subject will
   
    have a JOB_STOP enqueued 

Binds the unit to the object. Whenever the object enters the inactive or failed state, then a stop job for the subject will be triggered. If one uses this relationship without After=, then the start job will be dequeued from the manager's job queue and executed even when the object is in the *activating* state (mapping to what ever the sub state of the unit type is), however, on use of After=, the relationship is even stronger, in that the object has to enter the *active* state for the job for subject to be dequeued and executed. This will also mean that when stopping the subject has to stop fully before the object will be stopped, so it will strictly remain active only as long as the object unit is active.

This unit triggers propagation of job types when either the job is enqueued explicitly either by systemd, or when the unit itself causes a state change.

Those who are wondering how we were getting BindsTo= to work with Before=, since the man page explains that unit has to be active (or atleast activating without After=) for it to work? Without After=, BindsTo= checks are skipped. With Before=, we also disable failure propagation from that job by starting before it.

Remember that is there is job waiting for it in the transaction, if we use BindsTo= alone, there is chance that job completes and goes to inactive before we even start up (think oneshot units going straight from starting->inactive/failed), the unit_check_binds_to function has a check for u->job at the top causing true (one job slot per unit) to return. Now, by the time we are up, a check is triggered in the unit, and until a job is either up or the unit state isn't something other than UNIT_ACTIVE, we are good and won't stop. You are welcome :).

Note that this function is there all alone for BindsTo= to serve a very special property: BindsTo= is about binding oneself to the activation state of the other unit (though not totally true without After=). This means that for oneshot units that do not have RemainAfterExit= (and hence straight go from activating->inactive), BindsTo= will fail the starting unit. However, using job processors like Requires= would not (because while it goes from activating->inactive, that is considered successful). If InterruptedBy= is used with Requires=+After= on such a oneshot unit (with PartOf=), this certain property of BindsTo= is missed.

* PartOf=

When object
    
    has a JOB_STOP enqueued because of systemd
    has JOB_RESTART enqueued because of systemd (warning: doc is incorrect in this case)

Subject will
    
    have the same job enqueued (but as JOB_TRY_RESTART to not wake up deps)

When Object
    
    Has a JOB_RESTART enqueued by its own shutdown (so JOB_RESTART and then converted to JOB_START)

Subject will
    
    Nothing - use BindsTo= if you want local/implicit unit state changes to affect yours.
    manager triggered JOB_RESTART propagated as JOB_TRY_RESTART

This is a propagative relationship like others, and causes JOB_STOP and JOB_RESTART job types to be propagated to all subscribers, JOB_RESTART is covered because JOB_RESTART when running is internally equivalent to JOB_STOP but is tuned to become JOB_START immediately when the stop job returns the JOB_DONE job result. This means whenever something causes a stop job/restart job for the object unit to be triggered, the subject i.e. us will also get those jobs propagated to us. 

IMP: The difference from Requires= is requires only propagates JOB_STOP, this will propagate JOB_RESTART too, and also the fact that specifying this alone will NOT pull in units (man page is wrong, it is only similar to Requires= for JOB_STOP propagation, nothing else).

This is not triggered on units that do not enter the stopping state but abruptly transition to inactive or failed (notable example are device units, over which the job manager has no control). Hence, PartOf= on device units has no effect (because no STOP jobs or RESTART jobs get installed at all, that could pull us through walking ConsistsOf= in that object).

* Conflicts=
JOB_START queues a JOB_STOP on the Conflicted unit, and vice versa. without ordering, there is no gurantee what order jobs happen in.

* Before=, After=

Control what order jobs are queued, dequeued and executed in a transaction. They will also define how they wait on each other, as a consequence.

* OnFailure=

This is defined with respect to change in unit state, but with Restart=, so when the process itself shuts down or fails due to some errors and until all restarts as configured are tried, object unit listed here is not triggered.

* PropagatesReloadTo=, ReloadPropagatedFrom=

Requests made for reload (job type JOB_RELOAD) on this object are propagated in whatever direction one of these two directives
configure. Note that targets are kind of special here, in that if you specify any of these two in them, they can be reloaded, otherwise they can't. Note that JOB_RELOAD job types are not merged or coalesced, because that creates chances of the unit misses config file updates.

* StopWhenUnneeded=

Manual page gives the correct explanation, but this also has the effect of immediately stop a unit that nothing else requires even when the user explicitly asks for it to start through the bus.

...

Regaring specifiers, these are resolved at load time, but environment variables at job execution time.

* Restart=

Units are only restarted when they change their state on process exit, killing through signals, or reaching a timeout set for it. However, if a user explicitly asks for a unit to be stopped, or BindsTo= triggers it to be stopped (i.e. both being result of operation by the manager), the subject unit is not restarted. This is because Restart= only acts on implicit state changes, and not on explicit jobs causing a state change (and all of propagation dependencies are explicit i.e. enforced by systemd).

...

ExecStopPost= is run even when service fails, or due to any of the previous Exec*= commands failing, and also on success. This directive is excepted from RemainAfterExit= as it is run after the unit's stop/stop-sigterm/stop-terminate state is reached (as part of stop-final/stop-final-sigterm/stop-final-terminate), so this will not run for a unit with RemainAfterExit= (which is not useful for things that are not of Type=oneshot) but only when it is explicitly stopped.

...

Collapsing and tuning

JOB_TRY_RELOAD

     JOB_NOP if unit is inactive/deactivating
     JOB_RELOAD otherwise
     
JOB_TRY_RESTART

    JOB_NOP if unit is inactive/deactivating
    JOB_RESTART otherwise
    
JOB_TRY_RELOAD_OR_START

    JOB_START if unit is inactive/deactivating
    JOB_RELOAD otherwise
    
try-reload-or-restart bus operation

    A reload_if_possible bool with JOB_TRY_RESTART, if true and unit can reload, patch to become JOB_TRY_RELOAD, if false, keep JOB_TRY_RESTART as is.
    
reload-or-restart bus operation

    A reload_if_possible bool with JOB_RESTART, if true and unit can reload, patch to become JOB_RELOAD_OR_START, if false, keep JOB_RESTART as is.
    
try-restart bus operation

    A reload_if_possible bool with JOB_TRY_RESTART, set to false, collapse to JOB_NOP if unit is inactive/deactivating, otherwise queue a JOB_RESTART.


For targets, if reload propagation is enabled, they will return true for unit_can_reload, otherwise they only support restart. Job types will be tuned accordingly. Currently, there is no JOB_RELOAD_OR_RESTART job type (so reload_if_possible true makes it JOB_RELOAD for those who support it and JOB_RESTART for those who don't, or their try variants if so is requested).

Jobs only complete when manager matches against the new state, determines the unit is correctly in that state (and what it is supposed to be in) from the old state, and sets the job result (most likely to JOB_DONE if successful).
