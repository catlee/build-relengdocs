.. _Integrating-the-Cloud:

Integrating the Cloud
---------------------

* Software:

  + `Cloud Tools`_

* Repos:

  + https://hg.mozilla.org/build/cloud-tools/
  + https://hg.mozilla.org/build/puppet

* Purpose:

  + learn how watch pending works, what it does, why it exists.

aws_watch_pending.py_ is responsible for starting new instances on AWS to run pending jobs.

Generally, it is run periodically through cron::

   python aws_watch_pending.py -k secrets/aws-secrets.json -c ../configs/watch_pending.cfg -r us-west-2 -r us-east-1 \
    --cached-cert-dir secrets/cached_certs -l aws_watch_pending.log

The cron jobs are currently running on
aws-manager1.srv.releng.scl3.mozilla.com as the 'buildduty' user, and are managed by puppet in the
aws_manager_ module.

aws_watch_pending.py has this basic flow:

* Look at the set of pending jobs in the `scheduler database`_ via simple
  SQL query::

        SELECT buildername, id FROM
               buildrequests WHERE
               complete=0 AND
               claimed_at=0

* Map those to "instance types" given a set of regular expressions (yay!) in watch_pending.cfg_.
  Instance types are general classes of machines that correspond roughly to
  pools of slaves. For example, we have a "bld-linux64" type and "tst-linux64"
  type to do linux64-based builds and 64-bit tests, respectively. The
  instance type is recorded on the EC2 instances with the "moz-type" tag.

  For example::

    "buildermap": {
        "Android.* (?!try|Tegra|Panda|Emulator)\\S+ (build|non-unified)": "bld-linux64",
        "Android.* (?!Tegra|Panda|Emulator)try build": "try-linux64",
        "^Linux (?!try)\\S+ (pgo-|leak test )?build": "bld-linux64",
        "^Linux x86-64 (?!try)\\S+ (pgo-|leak test |asan |debug asan |debug static analysis )?(build|non-unified)": "bld-linux64",
        ...
    }

  Here we match various builder names from the pending jobs, and map them
  to the instance types. The builder names are generated as part of
  generateBranchObjects, and the entries in the database have already been
  added by the `buildbot schedulers`_.

  Jobs that have no matching instance type are assumed to be not runnable
  on AWS, and so we ignore them from here on.

* As we're finding jobs that have matching instance types, we keep a count
  of how many per instance type we need.

  We also look to see if the builder name is part of a jacuzzi_. If so,
  then it will have a specific set of slaves allocated to it, and we need
  to keep track of those in order to start the right machines.::

    # Mapping of (instance types, slaveset) to # of instances we want to
    # creates
    to_create = {
        'spot': defaultdict(int),
        'ondemand': defaultdict(int),
    }
    to_create_ondemand = to_create['ondemand']
    to_create_spot = to_create['spot']

    # Map pending builder names to instance types
    for pending_buildername, brid in pending:
        for buildername_exp, moz_instance_type in builder_map.items():
            if re.match(buildername_exp, pending_buildername):
                slaveset = get_allocated_slaves(pending_buildername)
                to_create_spot[moz_instance_type, slaveset] += 1
                break

    if not to_create_spot and not to_create_ondemand:
        return

* Now that we have a count of machines per instance type, we apply some
  heuristics to the count to prevent starting too many machines at once::

    for create_type, d in to_create.iteritems():
        for (moz_instance_type, slaveset), count in d.iteritems():
            running = aws_get_running_instances(all_instances, moz_instance_type)
            running = aws_get_slaveset_instances(running, slaveset)
            # Filter by create_type
            if create_type == 'spot':
                running = aws_get_spot_instances(running)
            else:
                running = aws_get_ondemand_instances(running)

            # Get instances launched recently
            fresh = aws_get_fresh_instances(running, time.time() - FRESH_INSTANCE_DELAY)
            num_fresh = len(fresh)

            num_old = len(running) - num_fresh
            delta = num_fresh + (num_old / 10)
            d[moz_instance_type, slaveset] = max(0, count - delta)

 * Reduce the number of required instances by 1 for every instance we've
   started recently. This timeout is defined as the FRESH_INSTANCE_DELAY_ and
   is currently set to 20 minutes. The reasoning here is that a machine can
   take many minutes to be fulfilled, boot up, run puppet, connect to
   buildbot, and finally be available to do jobs. Since aws_watch_pending
   is run every few minutes, we would otherwise be starting up too many
   instances in response to a single pending jobs.

 * Reduce the number of required instances by 10% of the number of running
   instances (excluding the "fresh" instances above). The reasoning here is
   that there is some chance that one of the running machines will complete
   its jobs and be available to take on more work before a new instance can
   start up and be available.




.. _cloud tools: https://hg.mozilla.org/build/cloud-tools/
