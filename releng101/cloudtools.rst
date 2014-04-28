Integrating the Cloud
---------------------

* Software:

  + `Cloud Tools`_

* Repos:

  + http://hg.mozilla.org/build/cloud-tools/
  + http://hg.mozilla.org/build/puppet

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
                log.debug("%s instance type %s slaveset %s", pending_buildername, moz_instance_type, slaveset)
                to_create_spot[moz_instance_type, slaveset] += 1
                break
        else:
            log.debug("%s has pending jobs, but no instance types defined",
                      pending_buildername)

    if not to_create_spot and not to_create_ondemand:
        log.debug("no pending jobs we can do anything about! all done!")
        return







