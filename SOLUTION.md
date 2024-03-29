# SOLUTION: Simudyne Hunger Games

## Baseline
There is a significant variation in total run time for executions using the same environment, code, scenario and settings. This is strange because the code uses a seed to generate randomness, so in principle it should have a deterministic run time. It happens both using it in Docker and from my local machine.

To work around this problem, I ran each code variation consecutively. Normally, the first time you get a longer run time, and after that it tends to converge. I run each variation 4 times.

I used 10,000 ticks as model length instead of the default 60 ticks to try to reduce the variation even more.

Each run was executed using `mvn clean compile exec:java`.

The values seen in the Console at the last step (10,000) pretty much remain constant:

**QUEUELENGTH**

*MEAN:* 5.95K

*HIGH:* 6K

*LOW:* 0

**NUMPRODUCTSDONE**

*MEAN:* 662.88

*HIGH:* 1.32K

*LOW:* 0

## Changes made

### v1: Group together all runs

I group together all the different segments comprising a `step()` in the `Factory` class.

Instead of this:

        run(
                // 1. machine finishes product -> push downstream AND flag upstream that you have space (must be in 1 func)
                Machine.pushDownstreamAndFlagUpstream(),

                // 2. conveyors: get products from upstream machine. Push out oldest if downstream is free (must be in 1 func)
                Conveyor.receiveProductAndPushOutOldest(),
                // 3. machine receives product from upstream for next tick
                Machine.receiveProductForWork()
        );

        run( // after adjusting conveyor queues, move all products by conveyor speed
                Conveyor.advanceAllProducts()
        );

        run( // let new products enter the system regularly
                Conveyor.addNewProducts()
        );

I do this:

        run(
                // 1. machine finishes product -> push downstream AND flag upstream that you have space (must be in 1 func)
                Machine.pushDownstreamAndFlagUpstream(),
                // 2. conveyors: get products from upstream machine. Push out oldest if downstream is free (must be in 1 func)
                Conveyor.receiveProductAndPushOutOldest(),
                // 3. machine receives product from upstream for next tick
                Machine.receiveProductForWork(),
                Conveyor.advanceAllProducts(),
                Conveyor.addNewProducts()
        );

### v2: Disable Unnecessary Loggers
In addition to changes of v1, I disabled loggers everywhere except the `Factory` class. This frees up CPU/Memory resources.

### v3: Discretization and Split

#### Discretization
In addition to the changes of v2, I applied step discretization, meaning that now each tick will be `K` times faster, where `K` is a parameter that is chosen so that there is a good balance between the end result (as measured by the output KPIs in the console) and run time. The benchmark is the number of products done when no discretization is applied `(K=1, NUMPRODUCTSDONE.HIGH=1.32K)`

Originally, I tried `K`=0.1/0.002223 = `45ms` (rounded from 44.9842...ms). The assumption was that since each product is 0.1m and the speed is 0.002223 m/ms, I could consider this as a basic discrete step.

However, using `K=45ms` did not work as I expected. That is because the minimum cycle time of a product in a machine is defined as `cycleTimeMin_ticks=10ms`, which means some products may take this amount of time to be machined. As a result, machines will not be able to notify upstream conveyors after 10ms (they would need to wait until the whole 45ms step is over). Those machines will be idle for a few milliseconds, and therefore, the factory will have a lower throughput overall. This was observed in the Console `(K=45, NUMPRODUCTSDONE.HIGH=903)`. Hence, we should at least reduce to `K=10ms`.

But still, using `K=10ms` did not produce an acceptable throughput: `(K=10, NUMPRODUCTSDONE.HIGH=1.21K)`. 1.21K products produced represents an 8% reduction with respect to the baseline (1.21K/1.32K=91.6%). Why? Since it takes ~45ms for a product to take the "slot" of the previous product in a conveyor, the total amount of time it will take for the product to walk across the conveyor will be 45ms * (9000m/0.1m) * 0.00223m/ms = 9003.15ms. This is not divisible by 10ms and it would cause the conveyor to be iddle (in some cases) for an additional 7ms. Hence, we came up with `K=5ms` which produces a throughput that is quite close to the benchmark `(K=5, NUMPRODUCTSDONE.HIGH=1.26K)` and reduces the execution time by roughly 1/5. This comes at a traded off cost of a tiny 4.5% reduction in throughput (1.26K/1.32K=95.45%).

**NOTE:** When using discretization, you should now divide the console **Model lenght (ticks)** by `K`. So, for `K=5` insted of using 10,000 ticks you should use 2,000 ticks.

#### Split

In this version, we additionally tried to introduce parallelization of two calls: `Machine.receiveProductForWork()` // `Conveyor.advanceAllProducts()` using `split()`. The intuition is that these two steps can be run independently and therefore in parallel, which should result in further reduction in run time.

        run(
                // 1. machine finishes product -> push downstream AND flag upstream that you have space (must be in 1 func)
                Machine.pushDownstreamAndFlagUpstream(),

                // 2. conveyors: get products from upstream machine. Push out oldest if downstream is free (must be in 1 func)
                Conveyor.receiveProductAndPushOutOldest(),
                // 3. machine receives product from upstream for next tick
                Split.create(Machine.receiveProductForWork(),
                Conveyor.advanceAllProducts()),
                Conveyor.addNewProducts()
        );

## Results
Below are the run times experienced on my computer (Apple M1 Pro 16GB Memory) using a `VARIANT=16-bullseye` Docker configuration, with the following resources allocated to Docker: 6 CPUs, 14GB RAM, 3GB Swap, 200GB Disk image size). **The reduction goal in run time was "~50%" and I was able to reduce it by 48% on v2 and 86% in v3 😌**.  

### v0: Baseline (10,000 ticks)
    2022-05-02 10:51:24,993 [INFO] [Simudyne-akka.actor.default-dispatcher-12] factory - Total Run time = 11879
    2022-05-02 10:53:20,701 [INFO] [Simudyne-akka.actor.default-dispatcher-6] factory - Total Run time = 8938
    2022-05-02 10:55:11,118 [INFO] [Simudyne-akka.actor.default-dispatcher-7] factory - Total Run time = 9102
    2022-05-02 10:57:53,346 [INFO] [Simudyne-akka.actor.default-dispatcher-11] factory - Total Run time = 8431

**Average:** 9587.5 ms

### v1: Group together all runs (10,000 ticks)
    2022-05-02 11:01:29,751 [INFO] [Simudyne-akka.actor.default-dispatcher-6] factory - Total Run time = 6305
    2022-05-02 11:03:53,962 [INFO] [Simudyne-akka.actor.default-dispatcher-12] factory - Total Run time = 5523
    2022-05-02 11:06:17,350 [INFO] [Simudyne-akka.actor.default-dispatcher-12] factory - Total Run time = 4553
    2022-05-02 11:08:17,317 [INFO] [Simudyne-akka.actor.default-dispatcher-13] factory - Total Run time = 5059

**Average:** 5360 ms

**Run Time reduction vs baseline:** 44.1%

### v2: Disable Unnecessary Loggers (10,000 ticks)
    2022-05-02 11:14:42,646 [INFO] [Simudyne-akka.actor.default-dispatcher-10] factory - Total Run time = 5808
    2022-05-02 11:18:33,369 [INFO] [Simudyne-akka.actor.default-dispatcher-6] factory - Total Run time = 4430
    2022-05-02 11:20:22,861 [INFO] [Simudyne-akka.actor.default-dispatcher-5] factory - Total Run time = 5034
    2022-05-02 11:38:14,455 [INFO] [Simudyne-akka.actor.default-dispatcher-12] factory - Total Run time = 4619

**Average:** 4972.8 ms

**Run Time reduction vs baseline:** 48.13%

### v3: Discretization and Split (K=5, 2,000 ticks)
        2022-05-03 05:52:41,745 [INFO] [Simudyne-akka.actor.default-dispatcher-16] factory - Total Run time = 1395
        2022-05-03 05:55:22,883 [INFO] [Simudyne-akka.actor.default-dispatcher-9] factory - Total Run time = 1386
        2022-05-03 05:56:01,509 [INFO] [Simudyne-akka.actor.default-dispatcher-10] factory - Total Run time = 1328
        2022-05-03 05:56:16,074 [INFO] [Simudyne-akka.actor.default-dispatcher-5] factory - Total Run time = 1344

**Average:** 1363.3ms

**Run Time reduction vs baseline:** 85.78%
