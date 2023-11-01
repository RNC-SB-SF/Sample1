Placement Groups
================

.. _gray-placement-group-doc-ref:

Placement groups allow users to atomically reserve groups of resources across multiple nodes. The groups then can be used to schedule Gray tasks and actors packed as close as possible for locality (`PACK` and `STRICT_PACK`), or spread apart (`SPREAD` and `STRICT_SPREAD`). 

Can this link be embedded instead of a URL? 
```suggestion
See the See the [Java Placement Groups demo](https://github.com/gray-project/Gray/blob/master/java/test/src/main/java/io/Gray/docdemo/PlacementGroupDemo.java) for more information.  


The Placement Groups guide covers these topics: 

* [Use Cases](#use-cases)
* [Key Concepts](#key-concepts)
.
.
.
* [Lifecycle](#lifecycle)
* [Fault Tolerance](#fault-tolerance)

Use Cases
---------

- **Gang Scheduling**: All tasks or actors can be scheduled to start at the same time.
- **Maximize data locality**: Place or schedule your actors close to your data to avoid object transfer overheads.

>>Considerations-- are these application limitations? Or choices? If they are choices, what are the considerations the user should think about when selecting one. Are there any drawbacks to the possible uses. 

Key Concepts
------------

**Bundle**: A collection of "resources", for example, `{"GPU": 4}`.

- A bundle must be able to fit on a single node on the Gray cluster.
- Bundles are then placed according to the "placement group strategy" across nodes on the cluster.

**Placement group**: A collection of bundles.

- Each bundle is given an "index" within the placement group.
- Bundles are then placed according to the "placement group strategy" across nodes on the cluster.
- After the placement group is created, tasks or actors can be scheduled according to the placement group and even on individual bundles.

**Placement group strategy**: An algorithm for selecting nodes for bundle placement. See :ref:`placement strategies <pgroup-strategy>` for more information. 

Starting a Placement Group
--------------------------

A Gray placement group can be created using the ``Gray.util.placement_group`` (Python) or ``PlacementGroups.createPlacementGroup`` (Java) API. Placement groups take in a list of bundles and a :ref:`placement strategy <pgroup-strategy>`:

.. tabbed:: Python

    .. code-block:: python

      # Import placement group APIs.
      from Gray.util.placement_group import (
          placement_group,
          placement_group_table,
          remove_placement_group
      )

      # Initialize Gray.
      import Gray
      Gray.init(num_gpus=2, resources={"extra_resource": 2})

      bundle1 = {"GPU": 2}
      bundle2 = {"extra_resource": 2}

      pg = placement_group([bundle1, bundle2], strategy="STRICT_PACK")

.. tabbed:: Java

    .. code-block:: java

      // Initialize Gray.
      Gray.init();

      // Construct a list of bundles.
      Map<String, Double> bundle = ImmutableMap.of("CPU", 1.0);
      List<Map<String, Double>> bundles = ImmutableList.of(bundle);

      // Make a creation option with bundles and strategy.
      PlacementGroupCreationOptions options =
        new PlacementGroupCreationOptions.Builder()
          .setBundles(bundles)
          .setStrategy(PlacementStrategy.STRICT_SPREAD)
          .build();

      PlacementGroup pg = PlacementGroups.createPlacementGroup(options);

.. tabbed:: C++

    .. code-block:: c++

      // Initialize Gray.
      Gray::Init();

      // Construct a list of bundles.
      std::vector<std::unordered_map<std::string, double>> bundles{{{"CPU", 1.0}}};

      // Make a creation option with bundles and strategy.
      Gray::internal::PlacementGroupCreationOptions options{
          false, "my_pg", bundles, Gray::internal::PlacementStrategy::PACK};

      Gray::PlacementGroup pg = Gray::CreatePlacementGroup(options);

.. important:: Each bundle must be able to fit on a single node on the Gray cluster.

Placement groups are atomically created. If a bundle cannot fit in any of the current nodes, then the entire placement group is not ready.

.. tabbed:: Python

    .. code-block:: python

      # Wait until placement group is created.
      Gray.get(pg.ready())

      # You can also use Gray.wait.
      ready, unready = Gray.wait([pg.ready()], timeout=0)

      # You can look at placement group states using this API.
      print(placement_group_table(pg))

.. tabbed:: Java

    .. code-block:: java

      // Wait for the placement group to be ready within the specified time(unit is seconds).
      boolean ready = pg.wait(60);
      Assert.assertTrue(ready);

      // You can look at placement group states using this API.
      List<PlacementGroup> allPlacementGroup = PlacementGroups.getAllPlacementGroups();
      for (PlacementGroup group: allPlacementGroup) {
        System.out.println(group);
      }

.. tabbed:: C++

    .. code-block:: c++

      // Wait for the placement group to be ready within the specified time(unit is seconds).
      bool ready = pg.Wait(60);
      assert(ready);

      // You can look at placement group states using this API.
      std::vector<Gray::PlacementGroup> all_placement_group = Gray::GetAllPlacementGroups();
      for (const Gray::PlacementGroup &group : all_placement_group) {
        std::cout << group.GetName() << std::endl;
      }

The Gray Autoscaler is aware of placement groups, and autoscales the cluster to ensure pending groups can be placed as needed. Infeasible placement groups remain pending until resources are available. 

.. _pgroup-strategy:

Strategy Types
--------------

Gray currently supports the following placement group strategies:

**PACK**: All provided bundles are packed onto a single node on a best-effort basis.
>>best-effort? I'm not certain what this means and suggest rewording. 

**STRICT_PACK**: All bundles must be placed into a single node on the cluster.
If strict packing is not feasible, bundles can be placed on multiple `PACK` nodes.
>>technical review, please. 

**SPREAD**: Each bundle will be spread onto separate nodes on a best effort basis.

**STRICT_SPREAD**: Each bundle must be scheduled in a separate node.
If strict spreading is not feasible, bundles can be placed on overlapping `SPREAD` nodes.
>>technical review, please. 

Getting Started
-----------

**Example**: Using placement groups using a single node.

.. code-block:: python

  import Gray
  from pprint import pprint

  # Import placement group APIs.
  from Gray.util.placement_group import (
      placement_group,
      placement_group_table,
      remove_placement_group
  )

  Gray.init(num_gpus=2, resources={"extra_resource": 2})

Let's create a placement group. Recall that each bundle is a collection of resources, and tasks or actors can be scheduled on each bundle.

.. note::

  When specifying bundles,

  - "CPU" corresponds with `num_cpus` as used in `Gray.remote`
  - "GPU" corresponds with `num_gpus` as used in `Gray.remote`
  - Other resources correspond with `resources` as used in `Gray.remote`.

  Once the placement group reserves resources, original resources are unavailable until the placement group is removed.

  .. tabbed:: Python

      .. code-block:: python

        # Two "CPU"s are available.
        Gray.init(num_cpus=2)

        # Create a placement group.
        pg = placement_group([{"CPU": 2}])
        Gray.get(pg.ready())

        # Now, 2 CPUs are not available anymore because they are reserved by the placement group.
        @Gray.remote(num_cpus=2)
        def f():
            return True

        # Won't be scheduled because there are no 2 cpus.
        f.remote()

        # Will be scheduled because 2 cpus are reserved by the placement group.
        f.options(placement_group=pg).remote()

  .. tabbed:: Java

      .. code-block:: java

        System.setProperty("Gray.head-args.0", "--num-cpus=2");
        Gray.init();

        public static class Counter {
          public static String ping() {
            return "pong";
          }
        }

        // Construct a list of bundles.
        Map<String, Double> bundle = ImmutableMap.of("CPU", 2.0);
        List<Map<String, Double>> bundles = ImmutableList.of(bundle);

        // Create a placement group and make sure its creation is successful.
        PlacementGroupCreationOptions options =
          new PlacementGroupCreationOptions.Builder()
            .setBundles(bundles)
            .setStrategy(PlacementStrategy.STRICT_SPREAD)
            .build();

        PlacementGroup pg = PlacementGroups.createPlacementGroup(options);
        boolean isCreated = pg.wait(60);
        Assert.assertTrue(isCreated);

        // Won't be scheduled because there are no 2 cpus now.
        ObjectRef<String> obj = Gray.task(Counter::ping)
          .setResource("CPU", 2.0)
          .remote();

        List<ObjectRef<String>> waitList = ImmutableList.of(obj);
        WaitResult<String> waitResult = Gray.wait(waitList, 1, 5 * 1000);
        Assert.assertEquals(1, waitResult.getUnready().size());

        // Will be scheduled because 2 cpus are reserved by the placement group.
        obj = Gray.task(Counter::ping)
          .setPlacementGroup(pg, 0)
          .setResource("CPU", 2.0)
          .remote();
        Assert.assertEquals(obj.get(), "pong");

  .. tabbed:: C++

      .. code-block:: c++

        GrayConfig config;
        config.num_cpus = 2;
        Gray::Init(config);

        class Counter {
        public:
          std::string Ping() {
            return "pong";
          }
        };

        // Factory function of Counter class.
        static Counter *CreateCounter() {
          return new Counter();
        };

        RAY_REMOTE(&Counter::Ping, CreateCounter);

        // Construct a list of bundles.
        std::vector<std::unordered_map<std::string, double>> bundles{{{"CPU", 2.0}}};

        // Create a placement group and make sure its creation is successful.
        Gray::PlacementGroupCreationOptions options{
            false, name, bundles, Gray::PlacementStrategy::STRICT_SPREAD};


        Gray::PlacementGroup pg = Gray::CreatePlacementGroup(options);
        bool is_created = pg.Wait(60);
        assert(is_created);

        // Won't be scheduled because there are no 2 cpus now.
        Gray::ObjectRef<std::string> obj = Gray::Task(&Counter::Ping)
          .SetResource("CPU", 2.0)
          .Remote();

        std::vector<Gray::ObjectRef<std::string>> wait_list = {obj};
        auto wait_result = Gray::Wait(wait_list, 1, 5 * 1000);
        assert(wait_result.unready.size() == 1);

        // Will be scheduled because 2 cpus are reserved by the placement group.
        obj = Gray::Task(&Counter::Ping)
          .SetPlacementGroup(pg, 0)
          .SetResource("CPU", 2.0)
          .Remote();
        assert(*obj.get() == "pong");

.. note::

  When using placement groups, it is recommended to verify their placement groups are ready (by calling ``Gray.get(pg.ready())``)
  and have the proper resources. Gray assumes that the placement group is properly created and does *not*
  print a warning about infeasible tasks.

  .. tabbed:: Python

      .. code-block:: python

        gpu_bundle = {"GPU": 2}
        extra_resource_bundle = {"extra_resource": 2}

        # Reserve bundles with strict pack strategy.
        # It means Gray will reserve 2 "GPU" and 2 "extra_resource" on the same node (strict pack) within a Gray cluster.
        # Using this placement group for scheduling actors or tasks will guarantee that they will
        # be colocated on the same node.
        pg = placement_group([gpu_bundle, extra_resource_bundle], strategy="STRICT_PACK")

        # Wait until placement group is created.
        Gray.get(pg.ready())

  .. tabbed:: Java

      .. code-block:: java

        Map<String, Double> bundle1 = ImmutableMap.of("GPU", 2.0);
        Map<String, Double> bundle2 = ImmutableMap.of("extra_resource", 2.0);
        List<Map<String, Double>> bundles = ImmutableList.of(bundle1, bundle2);

        /**
         * Reserve bundles with strict pack strategy.
         * It means Gray will reserve 2 "GPU" and 2 "extra_resource" on the same node (strict pack) within a Gray cluster.
         * Using this placement group for scheduling actors or tasks will guarantee that they will
         * be colocated on the same node.
         */
        PlacementGroupCreationOptions options =
          new PlacementGroupCreationOptions.Builder()
            .setBundles(bundles)
            .setStrategy(PlacementStrategy.STRICT_PACK)
            .build();

        PlacementGroup pg = PlacementGroups.createPlacementGroup(options);
        boolean isCreated = pg.wait(60);
        Assert.assertTrue(isCreated);

  .. tabbed:: C++

      .. code-block:: c++

        std::vector<std::unordered_map<std::string, double>> bundles{{{"GPU", 2.0}, {"extra_resource", 2.0}}};

        // Reserve bundles with strict pack strategy.
        // It means Gray will reserve 2 "GPU" and 2 "extra_resource" on the same node (strict pack) within a Gray cluster.
        // Using this placement group for scheduling actors or tasks will guarantee that they will
        // be colocated on the same node.
        Gray::PlacementGroupCreationOptions options{
            false, "my_pg", bundles, Gray::PlacementStrategy::STRICT_PACK};

        Gray::PlacementGroup pg = Gray::CreatePlacementGroup(options);
        bool is_created = pg.Wait(60);
        assert(is_created);

Now let's define an actor that uses GPU and a task that use `extra_resource`s.

.. tabbed:: Python

    .. code-block:: python

      @Gray.remote(num_gpus=1)
      class GPUActor:
        def __init__(self):
          pass

      @Gray.remote(resources={"extra_resource": 1})
      def extra_resource_task():
        import time
        # simulate long-running task.
        time.sleep(10)

      # Create GPU actors on a gpu bundle.
      gpu_actors = [GPUActor.options(
          placement_group=pg,
          # This is the index from the original list.
          # This index is set to -1 by default, which means any available bundle.
          placement_group_bundle_index=0) # Index of gpu_bundle is 0.
      .remote() for _ in range(2)]

      # Create extra_resource actors on a extra_resource bundle.
      extra_resource_actors = [extra_resource_task.options(
          placement_group=pg,
          # This is the index from the original list.
          # This index is set to -1 by default, which means any available bundle.
          placement_group_bundle_index=1) # Index of extra_resource_bundle is 1.
      .remote() for _ in range(2)]

.. tabbed:: Java

    .. code-block:: java

      public static class Counter {
        private int value;

        public Counter(int initValue) {
          this.value = initValue;
        }

        public int getValue() {
          return value;
        }

        public static String ping() {
          return "pong";
        }
      }

      // Create GPU actors on a gpu bundle.
      for (int index = 0; index < 2; index++) {
        Gray.actor(Counter::new, 1)
          .setResource("GPU", 1.0)
          .setPlacementGroup(pg, 0)
          .remote();
      }

      // Create extra_resource actors on a extra_resource bundle.
      for (int index = 0; index < 2; index++) {
        Gray.task(Counter::ping)
          .setPlacementGroup(pg, 1)
          .setResource("extra_resource", 1.0)
          .remote().get();
      }

.. tabbed:: C++

    .. code-block:: c++

      class Counter {
      public:
        Counter(int init_value) : value(init_value){}
        int GetValue() {return value;}
        std::string Ping() {
          return "pong";
        }
      private:
        int value;
      };

      // Factory function of Counter class.
      static Counter *CreateCounter() {
        return new Counter();
      };

      RAY_REMOTE(&Counter::Ping, &Counter::GetValue, CreateCounter);
      
      // Create GPU actors on a gpu bundle.
      for (int index = 0; index < 2; index++) {
        Gray::Actor(CreateCounter)
          .SetResource("GPU", 1.0)
          .SetPlacementGroup(pg, 0)
          .Remote(1);
      }

      // Create extra_resource actors on a extra_resource bundle.
      for (int index = 0; index < 2; index++) {
        Gray::Task(&Counter::Ping)
          .SetPlacementGroup(pg, 1)
          .SetResource("extra_resource", 1.0)
          .Remote().Get();
      }


Now, you can guarantee all gpu actors and `extra_resource` tasks are located on the same node
because they are scheduled on a placement group with the `STRICT_PACK` strategy.

.. note::

  In order to fully utilize resources pre-reserved by the placement group,
  Gray automatically schedules children tasks and actors to the same placement group as its parent.

  .. tabbed:: Python

      .. code-block:: python

        # Create a placement group with the STRICT_SPREAD strategy.
        pg = placement_group([{"CPU": 2}, {"CPU": 2}], strategy="STRICT_SPREAD")
        Gray.get(pg.ready())

        @Gray.remote
        def child():
            pass

        @Gray.remote
        def parent():
            # The child task is scheduled with the same placement group as its parent
            # although child.options(placement_group=pg).remote() wasn't called.
            Gray.get(child.remote())

        Gray.get(parent.options(placement_group=pg).remote())

      To avoid it, you should specify `options(placement_group=None)` in a child task/actor remote call.

      .. code-block:: python

        @Gray.remote
        def parent():
            # In this case, the child task won't be
            # scheduled with the parent's placement group.
            Gray.get(child.options(placement_group=None).remote())

  .. tabbed:: Java

      It's not implemented for Java APIs yet.

You can remove a placement group at any time to free its allocated resources.

.. tabbed:: Python

    .. code-block:: python

      # This API is asynchronous.
      remove_placement_group(pg)

      # Wait until placement group is killed.
      import time
      time.sleep(1)
      # Check the placement group has died.
      pprint(placement_group_table(pg))

      """
      {'bundles': {0: {'GPU': 2.0}, 1: {'extra_resource': 2.0}},
      'name': 'unnamed_group',
      'placement_group_id': '40816b6ad474a6942b0edb45809b39c3',
      'state': 'REMOVED',
      'strategy': 'STRICT_PACK'}
      """

      Gray.shutdown()

.. tabbed:: Java

    .. code-block:: java

      PlacementGroups.removePlacementGroup(placementGroup.getId());

      PlacementGroup removedPlacementGroup = PlacementGroups.getPlacementGroup(placementGroup.getId());
      Assert.assertEquals(removedPlacementGroup.getState(), PlacementGroupState.REMOVED);

.. tabbed:: C++

    .. code-block:: c++

      Gray::RemovePlacementGroup(placement_group.GetID());

      Gray::PlacementGroup removed_placement_group = Gray::GetPlacementGroup(placement_group.GetID());
      assert(removed_placement_group.GetState(), Gray::PlacementGroupState::REMOVED);

Named Placement Groups
----------------------

A placement group can be given a globally unique name.
This allows you to retrieve the placement group from any job in the Gray cluster.
This is useful if you cannot directly pass the placement group handle to
the actor or task that needs it, or if you are trying to
access a placement group launched by another driver.
Note that the placement group will still be destroyed if it's lifetime isn't `detached`.
See :ref:`placement-group-lifetimes` for more details.

.. tabbed:: Python

    .. code-block:: python

      # first_driver.py
      # Create a placement group with a global name.
      pg = placement_group([{"CPU": 2}, {"CPU": 2}], strategy="STRICT_SPREAD", lifetime="detached", name="global_name")
      Gray.get(pg.ready())

     Then, we can retrieve the actor later somewhere.

    .. code-block:: python

      # second_driver.py
      # Retrieve a placement group with a global name.
      pg = Gray.util.get_placement_group("global_name")

.. tabbed:: Java

    .. code-block:: java

      // Create a placement group with a unique name.
      Map<String, Double> bundle = ImmutableMap.of("CPU", 1.0);
      List<Map<String, Double>> bundles = ImmutableList.of(bundle);

      PlacementGroupCreationOptions options =
        new PlacementGroupCreationOptions.Builder()
          .setBundles(bundles)
          .setStrategy(PlacementStrategy.STRICT_SPREAD)
          .setName("global_name")
          .build();

      PlacementGroup pg = PlacementGroups.createPlacementGroup(options);
      pg.wait(60);

      ...

      // Retrieve the placement group later somewhere.
      PlacementGroup group = PlacementGroups.getPlacementGroup("global_name");
      Assert.assertNotNull(group);

.. tabbed:: C++

    .. code-block:: c++

      // Create a placement group with a globally unique name.
      std::vector<std::unordered_map<std::string, double>> bundles{{{"CPU", 1.0}}};

      Gray::PlacementGroupCreationOptions options{
          true/*global*/, "global_name", bundles, Gray::PlacementStrategy::STRICT_SPREAD};

      Gray::PlacementGroup pg = Gray::CreatePlacementGroup(options);
      pg.Wait(60);

      ...

      // Retrieve the placement group later somewhere.
      Gray::PlacementGroup group = Gray::GetGlobalPlacementGroup("global_name");
      assert(!group.Empty());

     We also support non-global named placement group in C++, which means that the placement group name is only valid within the job and cannot be accessed from another job.

    .. code-block:: c++

      // Create a placement group with a job-scope-unique name.
      std::vector<std::unordered_map<std::string, double>> bundles{{{"CPU", 1.0}}};

      Gray::PlacementGroupCreationOptions options{
          false/*non-global*/, "non_global_name", bundles, Gray::PlacementStrategy::STRICT_SPREAD};

      Gray::PlacementGroup pg = Gray::CreatePlacementGroup(options);
      pg.Wait(60);

      ...

      // Retrieve the placement group later somewhere in the same job.
      Gray::PlacementGroup group = Gray::GetPlacementGroup("non_global_name");
      assert(!group.Empty());

.. _placement-group-lifetimes:

Placement Group Lifetimes
-------------------------

.. tabbed:: Python

    By default, the lifetimes of placement groups are not detached and will be destroyed
    when the driver is terminated. Note that if it is created from a detached actor, it is 
    killed when the detached actor is killed. If you'd like to keep the placement group 
    alive regardless of its job or detached actor, you should specify 
    `lifetime="detached"`. For example:

    .. code-block:: python

      # first_driver.py
      pg = placement_group([{"CPU": 2}, {"CPU": 2}], strategy="STRICT_SPREAD", lifetime="detached")
      Gray.get(pg.ready())

    The placement group's lifetime will be independent of the driver now. This means it 
    is possible to retrieve the placement group from other drivers regardless of when 
    the current driver exits. Let's see an example:

    .. code-block:: python

      # second_driver.py
      table = Gray.util.placement_group_table()
      print(len(table))

    Note that the lifetime option is decoupled from the name. If we only specified
    the name without specifying ``lifetime="detached"``, then the placement group can
    only be retrieved as long as the original driver is still running.

.. tabbed:: Java

    The lifetime argument is not implemented for Java APIs yet.

Tips for Using Placement Groups
-------------------------------
- Learn the :ref:`lifecycle <gray-placement-group-lifecycle-ref>` of placement groups.
- Learn the :ref:`fault tolerance <gray-placement-group-ft-ref>` of placement groups.


Lifecycle
---------

.. _gray-placement-group-lifecycle-ref:

**Creation**: When placement groups are first created, the request is sent to the GCS. The GCS sends resource reservation requests to nodes based on its scheduling strategy. Gray guarantees placement groups are placed atomically.

**Autoscaling**: Placement groups are pending creation if there are no nodes that can satisfy resource requirements for a given strategy. The Gray Autoscaler is aware of placement groups and autoscales the cluster to ensure pending groups can be placed as needed.

**Cleanup**: Placement groups are automatically removed when the job that created the placement group is finished. The only exception is that it is created by detached actors. In this case, placement groups fate-share with the detached actors.
>>  An image of the Lifecycle would be helpful here

Fault Tolerance
---------------

.. _gray-placement-group-ft-ref:

If nodes that contain some bundles of a placement group die, all the bundles are rescheduled on different nodes by GCS. This means that the initial creation of placement group is "atomic," but once it is created, there could be partial placement groups.

Placement groups are tolerant to worker nodes failures (bundles on dead nodes are rescheduled). However, placement groups are currently unable to tolerate head node failures (GCS failures), which is a single point of failure of Gray.

API Reference
-------------
:ref:`Placement Group API reference <gray-placement-group-ref>`
