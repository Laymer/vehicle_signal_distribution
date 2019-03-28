 # VEHICLE SIGNAL DISTRIBUTION [VSD]
Library to publish and subscribe Vehicle Signal Specification signals
via reliable multicast.


VSD reads a signal specification CSV file, generated from the GENIVI
Vehicle Signal Specification project, and provides API calls to
set, publish, subscribe, and receive to these signals.

Please see [Vehicle Signal Specification (VSS)](https://github.com/GENIVI/vehicle_signal_specification)
project for details on branches, signal structures and attributes.

Below is an overview of how signals are structured.
![VSS Overview](illustrations/vsd_overview.png "Vehicle Signal Specification Overview")
*Fig. 1. Vehicle Signal Specification Overview*

**Branches** host other branches and signals and form an overall signal structure.<br>
**Sensors** contain signals read from actual sensors and signals
calculated from multiple sensor sources.<br>
**Attributes** are static values containing configuration-level data.<br>

A signal\'s value is set and published as two separate operations.

Publishing can be done on an individual signal level, or on a branch
level. If a branch is published, the branch and all its children are
published in an atomic operation. This allows complex signal
structures to be easily transmitted.

Subscriptions can also be done an signal or on a branch level. If a
branch is subscribed to, a callback will be made for any signal that
is updated under that branch.

## Using Vehicle Signal Distribution:
Be sure that you have built and deployed libraries for both dstc and
reliable_multicast (RMC), as they are dependencies of the vsd project.

The libraries can be found at:

[Reliable Multicast v1.3](https://github.com/PDXostc/reliable_multicast/releases/tag/v1.3)<br>
[DSTC v1.2](https://github.com/PDXostc/dstc/releases/tag/v1.2)<br>

Build and install RMC first, followed by DSTC, and ensure that
`Makefile` has include and link paths setup to the install
directories.

VSD currently will produce a shared object to link against, and also
compiles two example programs to begin working with the project. Build
and install these using:

    make
    make DESTDIR=/usr/local install
    make examples
    make DESTDIR=/usr/local install_examples

## RUNNING THE EXAMPLE
The programs `vsd_pub_example` and `vsd_sub_examples` are built and
installed, providing an insight into how VSD works.

## Running `vsd_sub_example`
The usage for `vsd_sub_example`, which subscribes to signals sent
by `vsd_pub_example` is shown below.

     ./examples/vsd_sub_example /usr/local/share/vss_rel_2.0.0-alpha+005.csv Vehicle.Drivetrain.InternalCombustionEngine

The first argument, `/usr/local/share/vss_rel_2.0-alpha+005.csv` is a
CSV variant of the vehicle signal specification generated by running
`make` in a checked out VSS repo. It contains a tree structure of all
signals, their type, allowed values, if they are actuators, sensors,
or others, etc. VSD uses this as a global specification of all signals
that can possibly be seen on the network, even if not all of them are
supported in a given deployment.

All VSD nodes must load the same version of the file in order to have
the definition of signals.

A sample VSS csv file is installed under the `share` directory of the
VSD install directory.

The second argument, `Vehicle.Drivetrain.InternalCombustionEngine`,
specifies the subtree that is to be subscribed to. If any signal in
the given subtree is received, a callback will be made to the
subscriber. This allows the subscriber to chose the granularity of the
subscription, from individual signals to the whole vehicle state.

## Running `vsd_pub_example`
The usage for `vsd_pub_example`, which publishes signals to
zero or more `vsd_sub_example` instances network, is shown below.

    ./examples/vsd_pub_example -d /usr/local/share/vss_rel_2.0.0-alpha+005.csv \
                    -s Vehicle.Drivetrain.InternalCombustionEngine.Engine.Power:230 \
                    -s Vehicle.Drivetrain.InternalCombustionEngine.FuelType:gasoline \
                    -p Vehicle.Drivetrain.InternalCombustionEngine

The `-d /usr/local/share/vss_rel_2.0.0-alpha+005.csv` argument
specifies where to load the signal specification file from. This file
must be the same as that used by `vsd_sub_example`.

The `-s Vehicle.Drivetrain.InternalCombustionEngine.Engine.Power:230`
argument sets the value of the `Power` signal for the engine to `230`.

The `-s Vehicle.Drivetrain.InternalCombustionEngine.FuelTyoe:gasoline`
argument sets the value of the `FuelType` signal for the engine to `gasoline`.

The `-p Vehicle.Drivetrain.InternalCombustionEngine`
argument specifies that all signals under the `InternalCombustionEngine` should be published
atomically.

Atomic signal publishing allows for the transmission of arbitrarily
complex signals as a single, cohesive unit.  The callback will receive
a list of all the published signals, which represents a snapshot of
the published tree at that exact moment in time.

## API CALL FLOW - PUBLISHER
The call flow for the publisher is illustrated below. Please note that
the calls shown have slight name and argument variations in the
implementation. Please see the sample code in `examples/vsd_pub_example.c` and
`examples/vsd_sub_example.c` for further details.

The signals this example are simplified down to two sensors, RPM
(engine speed), ECT (engine coolant temperature), and one attribute,
FuelType.
![Publisher overview](illustrations/vsd_pub_1.png "Publisher Overview")
*Fig. 2. Signal Publisher overview*

### Loading VSS signal descriptor file

The VSD system is hosted by a context variable that is setup by the library.
In order to initialize VSD, a pointer to a `vsd_context_t` pointer is provided
to `vsd_load_from_file()` together with the VSS file to load.

    vsd_context_t* ctx = 0;
    vsd_load_from_file(&ctx, "vss_2.0.0.csv");

The `vss_2.0.0.csv` file is generated by the VSS project (link in
introduction). Please note that the loaded CSV file needs to have
signal IDs for branches, not only the signals themselves.  To verify
if this is the case, check that the second field in the CSV has a
number (in quotes) for each line.

The provided `ctx` pointer will be set to an internal context. All
subsequent signal operations will use this pointer as an argument.

### Setting the first signal

The signal publisher starts the process of distributing updated signal
values to subscribers by setting a signal through a VSD call:

![Publisher step 1](illustrations/vsd_pub_2.png "Setting the first signal")
*Fig. 3. Setting the first signal*

A signal value can be set by its name (slow), its unique signal ID
integer (less slow), or by a signal descriptor retrieved by name or ID
(fast). In the example above we set the signal by name, which is a
complete VSS path to the signal.

The signal value is changed from its original 2350 to 2400.

### Setting the second signal
Additional signals can subsequently be set by the publisher:

![Publisher step 2](illustrations/vsd_pub_3.png "Setting the second signal")
*Fig. 4. Setting the second signal*

In this case we change ECT from 89 to 94 degrees (Centigrade). We can
set an arbitrary number of signals, and also set the same signal
multiple times.


### Publishing a signal subtree
Once all signals have been updated they can be published. Publishing is can be done on an individual signal level or on a branch level as shown below.

![Publisher step 3](illustrations/vsd_pub_4.png "Publishing signals")
*Fig. 5. Publishing signals*

In this case all signals hosted by the Engine branch, ECT, RPM, and
FuelType, will be published. Please note that FuelType is included
although it has not been updated.

*Future improvement: There will be an option to specify
if unchanged signals should be published or not.*

An individual signal can be published by simply specifying the full
VSS path to it:

    vsd_publish("Engine.RPM");

Multiple nodes in a network can publish the same signals.

Internally, the signals are published via the `signal_transmit()` DSTC RPC call. This call
will be executed by all nodes that are using VSD, and thereby implements `signal_transmit()` as
a DSTC server function.

*Future improvement: Additions will be made to query initial state of signal, query which
signals are currently being published, and default value in case a
signal is not published by any node in the network.*


### Processing DSTC events to transmit data
VSD uses DSTC (and its underlying Reliable Multicast) for all network
traffic, and DSTC event processing calls are used to receive and transmit
signal-carrying UDP multicast packets over the network.
In order to transmit the data, `dstc_process_events()` is called.

![Publisher step 4](illustrations/vsd_pub_5.png "Processing events")
*Fig. 6. Processing events*

Please see the DSTC repo's `examples` directory for different
variations of event processing, including moving the event loop out of
DSTC to the calling program.

### Transmitting published signals over the network.
The DSTC event processor will pick up the pending publish operation,
create a network packet and transmit it via the UDP multicast socket.

![Publisher step 5](illustrations/vsd_pub_6.png "Transmitting signals")
*Fig. 7. Transmitting signals*

The signal paths in the Fig. 7 packet are for illustration purposes
only. The actual packet uses a 32-bit numeric signal ID loaded from
the VSS file to identify the updated signal. The data is transmitted
as a little-endian-formatted binary scalar or a tagged-length string.

## API CALL FLOW - SUBSCRIBER
The call flow for the subscriber is illustrated below.

### Loading VSS signal descriptor file
The subscriber loads the CSS CSV file in the same way as the publisher.

### Subscribing to a subtree
Much like publishing, subscription can be done on a per-signal level
or on whole subtrees hosted by a branch. Subscriptions works by
setting up a callback to be invoked when one or more signals in the
subscribed-to tree (or individual signals) are updated through a
publish operation somewhere in the network.

![Subscriber step 1](illustrations/vsd_sub_1.png "Setting up subscription")
*Fig. 8. Setting up a subscription*

When one or more signals are received from an atomic publish of a
branch or specific signal, all subscription callbacks registered for
the published branch/signal will be invoked. Subscriptions for
branches higher up in the tree will also be invoked.

As an effect. If the root branch is subscribed to, all published
signals received by the VSD system will trigger a callback to that
Process.

Signals can be subscribed to by the same process that publishes
them. In these cases VSD will behave exactly as if the signals were
received from the network. In other words, the behavior is identical
for how locally and remotely published signals are processed.

### Process events
In order to receive and process published signals from the network,
the subscribing process must call `dstc_process_events()` in the same
way that the publisher does.

![Subscriber step 2](illustrations/vsd_sub_2.png "Setting up subscription")
*Fig. 9. Setting up a subscription*

### Receiving published signals
The DSTC event processor will read, unpack, and validate the published
signals read from the network.

![Subscriber step 3](illustrations/vsd_sub_3.png "Setting up subscription")
*Fig. 10. Setting up a subscription*

Internally the signals are received as a DSTC call to
`signal_transmit()` which is the C function that will get invoked by
the subscriber process.

### Updating signal values
The received signals are traversed and the internal signal tree
(maintained by `ctx` described above) will have its `ECT` and `RPM`
values updated in order to reflect the new signal state.

![Subscriber step 2](illustrations/vsd_sub_4.png "Setting up subscription")
*Fig. 11. Setting up a subscription*

`FuelType` will be marked as updated but have its value unchanged.

### Invoke callbacks
The VSD system will use the signal or branch published as a starting
point in the local tree in order to find subscribers where any subscription
callbacks registered at that signal/branch will be invoked.

![Subscriber step 2](illustrations/vsd_sub_5.png "Setting up subscription")
*Fig. 12. Invoking callbacks.

Once the immediate subscriber callbacks have been invoked, the parents
for the published signal/branch will be traversed upward toward the
signal tree root. Any subscription callbacks registered to these
parent branches will be invoked as well.
