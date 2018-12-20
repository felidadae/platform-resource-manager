Demo scenarios
....................
This document describes how to run demo scenarios on a Apache Mesos/Aurora cluster.
The commands included in the file allows to run a choosen demo scenario on a single pair of nodes.

Assumptions:

- working directory: PRM project top level directory,
- copied and filled the template file `demo_scenarios/common/common.template.yaml` to `demo_scenarios/common/common.yaml`,
- the nodes contains owca.pex file with PRM module included.


1. Export required environment variables. The variable `REMOTE_IP` should be set to
   one entry of the hosts defined in `demo_scenarios/common/common.yaml.`.

.. code-block:: sh

    export cinventory=demo_scenarios/common/common.template.yaml # inventory with list of hosts
                                                                 #  please copy template file and fill hosts
    export REMOTE_IP=<remote_ip> # one of the hosts defined in common/common.yaml
    export REMOTE_USER=<user>    # the ad-hoc ansible commands on $REMOTE_IP machine will be run
                                 #  by the user $REMOTE_USER
    export playbook=owca/workloads/run_workloads.yaml # path to owca run_workloads playbook



2. Kill all jobs, stop agent and remove a temporary model data:

.. code-block:: sh

    ansible-playbook -l $REMOTE_IP -i $cinventory $playbook --tags=clean_jobs -v
    ansible -u $REMOTE_USER -b all -i $REMOTE_IP, -msystemd -a'name=owca state=stopped'
    ansible -u $REMOTE_USER -b all -i $REMOTE_IP, -a'rm -f /var/lib/owca/lc-util.csv /var/lib/owca/workload-meta.json /var/lib/owca/workload-data.csv /var/lib/owca/threshold.json'



3. Start application.

.. code-block:: sh

    # simple scenario
    ansible-playbook -l $REMOTE_IP -i demo_scenarios/simple.0/inventory.yaml -i $cinventory $playbook \
      --tags=clean_jobs,twemcache_mutilate,tensorflow_benchmark_prediction,cassandra_stress--cassandra

    # or complex_llc.0
    ansible-playbook -l $REMOTE_IP -i demo_scenarios/complex_llc.0/inventory.yaml -i $cinventory $playbook \
      --tags=twemcache_mutilate,redis_rpc_perf,cassandra_stress--cassandra

    # or complex_mbw.0
    ansible-playbook -l $REMOTE_IP -i demo_scenarios/complex_mbw.0/inventory.yaml -i $cinventory $playbook \
      --tags=specjbb,tensorflow_benchmark_prediction,tensorflow_benchmark_training,cassandra_stress



4. Reconfigure OWCA (by setting `action_delay` to 20 seconds) and restart it to run in `collect` mode

.. code-block:: sh

    ansible -u $REMOTE_USER -b all -i $REMOTE_IP, -mreplace -a'path=/etc/owca/owca_config.yml after=detector regexp="detect" replace="collect"'
    ansible -u $REMOTE_USER -b all -i $REMOTE_IP, -mreplace -a'path=/etc/owca/owca_config.yml regexp="action_delay: .*" replace="action_delay: 20."'
    ansible -u $REMOTE_USER -b all -i $REMOTE_IP, -msystemd -a'name=owca state=restarted'



5. Keep creating and killing jobs to observe anomalies.

.. code-block:: sh

    # simple scenario
    ansible-playbook -l $REMOTE_IP -i demo_scenarios/simple.0/inventory.yaml -i $cinventory $playbook --tags=cassandra_stress--stress -v
    ansible-playbook -l $REMOTE_IP -i demo_scenarios/simple.0/inventory.yaml -i $cinventory $playbook --tags=clean_jobs -ekill_job_name=cassandra_stress.default--cassandra_stress--9142.0 -v

    # complex_llc.0
    ansible-playbook -l $REMOTE_IP -i $inventory $playbook --tags=cassandra_stress--stress -v
    ansible-playbook -l $REMOTE_IP -i $inventory $playbook --tags=clean_jobs -ekill_job_name=cassandra_stress.default--cassandra_stress--9142.0 -v

    # complex_mbw.0
    ansible-playbook -l $REMOTE_IP -i $inventory $playbook --tags=tensorflow_benchmark_prediction -v
    ansible-playbook -l $REMOTE_IP -i $inventory $playbook --tags=clean_jobs -ekill_job_name=tensorflow_benchmark_prediction.default--0.0 -v



Oneliners to check owca status, owca configuration file and size of PRM state files: 

.. code-block:: sh

    # to check status
    ansible -u $REMOTE_USER -b all -i $REMOTE_IP, -a'cat /etc/owca/owca_config.yml'
    ansible -u $REMOTE_USER -b all -i $REMOTE_IP, -a'systemctl status owca'
    ansible -u $REMOTE_USER -b all -i $REMOTE_IP, -a'ls -la /var/lib/owca/'
