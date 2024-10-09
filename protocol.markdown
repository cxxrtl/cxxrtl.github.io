---
layout: document
---

Version 0.14.0

```danger
This specification is in early stages of development and incompatible changes will be made without prior notice or backwards compatibility.
```

<!--
TODO:
- in Query Interval, `diagnostics` should take a list of types
- in Query Interval, `items` should be a reference or a list of references
- in all commands, `src` should be a list of source locations, and a source location should adhere to a regex
- introduce vendor keys with `domain.name!key` format
- 2-valued/4-valued/9-valued logic
- translation table for enums
- annotations
- allow omitting trailing 0's in time points?
-->

# CXXRTL debug server concepts

The *CXXRTL debug server* is a time series database specialized for storing and retrieving waveform data for digital simulations. The executable code implementing the simulation is used as a form of compression to greatly reduce the amount of space required to store the complete waveform data for a simulation. Because of this design choice, the processes of running the simulation and retrieving the waveform data are closely coupled.

From a client's point of view, the debug server can be conceptually split into two nearly independent interfaces:
* The simulation interface.
* The database query interface.

The simulation interface allows the client to start and stop the simulation, populating the waveform database with the data of interest while limiting resource consumption (e.g. by not simulating past the point of interest). The simulation may also be paused by the debug server itself on certain events (e.g. an assertion failure). Internally, the waveform database contains a replay buffer that sequentially lists each change to the design state in a format that allows bidirectional navigation, and running the simulation populates this replay buffer.

The database query interface allows the client to retrieve the waveforms and diagnostic messages produced by the simulation. Internally, the debug server will rewind the simulation to an earlier point in the replay buffer and re-run the simulation for the requested time interval while capturing the diagnostics and the waveforms for the requested items.

The above description of the internal operation of the debug server is provided for informational purposes. These internal functions are abstracted away in the debug server protocol, and are invoked implicitly in response to high-level client commands. An understanding of the internal operation nevertheless may be useful to implement a responsive and resource-efficient client.


# CXXRTL debug server protocol

The CXXRTL debug server and a debugging client interact by exchanging *messages* according to well-known rules called the *debug server protocol*. The debug server protocol does not define how the messages are transported, but is defined to be easily transported within byte-oriented streams, such as TCP sockets, Unix domain sockets, and Win32 named pipes.

```note
The CXXRTL debug server protocol may be implemented by any tools that wish to exchange waveform data. Its use is generally encouraged, and not restricted to the Yosys CXXRTL backend and the tools that interact with it.

There will eventually be a governance process for extending the protocol; while it remains at version 0, the steward of the protocol is [Catherine "whitequark"](mailto:whitequark@whitequark.org).
```

Each message is a UTF-8 string followed by the U+0000 *terminating character*, which is not a part of the message. (A message may not contain the terminating character within it.)

The contents of each message is a [JSON](https://json.org) object, whose structure is defined by the debug server protocol as described in the sections below.

The following abbreviations are used:
* S>C: message sent by Server to Client
* C>S: message sent by Client to Server

The structure of a message is defined by the *message type*, identifying its unique purpose and allowing the server to send asynchronous notifications in presence of pipelined commands.


## Message types

The debug server protocol defines a small amount of message types that define the overall framing. The message type defines how the message relates to other messages in the data stream, and the basic structure of the message, but delegates definition of the message semantics to the particular commands or events.


### Message type: *Greeting*

Whenever the client opens a new connection, it sends a *Greeting* message:

C>S:

```json
{
    "type": "greeting",
    "version": 0
}
```

The server responds with its capabilities:

S>C:

```json
{
    "type": "greeting",
    "version": 0,
    "commands": [
        "list_scopes",
        "list_items",
        "reference_items",
        "query_interval",
        "get_simulation_status",
        "run_simulation",
        "pause_simulation"
    ],
    "events": [
        "simulation_paused",
        "simulation_finished"
    ],
    "features": {
        "item_values_encoding": ["base64(u32)"]
    }
}
```

The `version` is fixed at 0.

The `commands` and `events` fields are arrays of strings that indicate which commands the server supports, and which events it will generate. Commands and events will not be changed in backwards-incompatible ways in this protocol (rather, a new command or event with a name ending in `_v2`, `_v3`, ... will be defined). Clients may use this information to support multiple versions of servers if desired.

The `features` field is a dict indicating which *protocol features* the server supports. Only the `item_values_encoding` protocol feature, listing the `item_values` encodings that can be requested in a *Query Interval* command, is currently supported.

Opening or closing the connection to the server does not affect the simulation status. However, events emitted while there is no open connection to the server are lost.

````note

As the protocol is intended for general use, in some cases only a subset of the features may be desirable to implement. For example, a waveform data post-processor has no means to run a simulation. In its greeting message, it should advertise the lack of simulation capabilities:

```json
{
    "type": "greeting",
    "version": 0,
    "commands": [
        "list_scopes",
        "list_items",
        "reference_items",
        "query_interval",
    ],
    "events": [],
    "features": {
        "item_values_encoding": ["base64(u32)"]
    }
}
```
````


### Message type: *Command*

Whenever the client needs the server to perform an operation, it sends a *Command* message:

C>S:

```json
{
    "type": "command",
    "command": "<command name>",
    "<argument name>": "<argument value>"
}
```

A command can have any amount of arguments with any names other than `type` and `command`, and they may be of any type. The structure of the command message is determined by the command name.

Whenever the server receives a command, it replies with either a *Response* or an *Error* message.


### Message type: *Response*

Whenever the server successfully processes a *Command* sent to it by the client, it sends a *Response* message with the identical command name:

S>C:

```json
{
    "type": "response",
    "command": "<command name>",
    "<argument name>": "<argument value>"
}
```

A response can have any amount of arguments with any names other than `type` and `command`, and they may be of any type. The structure of the response message is determined by the command name.

```note
The command name is duplicated in this message to aid debugging, and to allow validating individual response messages using a schema.
```


### Message type: *Error*

Whenever the server cannot recognize a *Command* sent to it by the client, or fails to process the *Command*, it sends a *Error* message:

S>C:

```json
{
    "type": "error",
    "error": "<error name>",
    "<argument name>": "<argument value>",
    "message": "<human readable description of the error>"
}
```

An error can have any amount of arguments with any names other than `type`, `error`, and `message`, and they may be of any type. The structure of the error message is determined by the error name.

The server does not terminate the connection on error. Further commands can be sent and processed normally.


### Message type: *Event*

In certain states, the server continues to execute code according to a *Command* message after sending a *Response* message. To notify the client of actions asynchronously performed by such code, the server may send a *Event* message. The client should be ready to process any number of *Event* messages at any time, including when it is expecting a *Response* message.

S>C:

```json
{
    "type": "event",
    "event": "<event name>",
    "<argument name>": "<argument value>"
}
```

An event can have any amount of arguments with any names other than `type` and `event`, and they may be of any type. The structure of the event message is determined by the event name.


## Commands

*Commands* are in effect remote procedure calls, enabling the client to manipulate the server state and receive confirmation of success or retrieve data.

Commands may be pipelined, meaning that the client may send several commands back-to-back, without waiting for the first command to complete to initiate the second command, etc. The server will respond to the pipelined commands in sequence. If one of the pipelined commands fails, it will continue processing the rest as usual, so pipelining should be restricted to those commands where it can be known in advance that they will not fail when invoked with the particular arguments. The protocol is designed to enable pipelining wherever feasible.


### Command: *List Scopes*

The *List Scopes* command allows the client to request the list of scopes and their descriptions. A scope is a unit of design hierarchy that contains debug items. A scope has an identifier, a type, and type-specific information. The identifier of a scope is a string. There are three kinds of scope identifiers:

* an empty string: identifier of the root scope.
* a string consisting of any characters except for `\u0020`: identifier of a scope nested within the root scope.
* a scope identifier followed by `\u0020` followed by a string consisting of any characters except for `\u0020`: identifier of a scope nested within another (non-root) scope.

Per the rules above, there can only be one `\u0020` character in a row in a scope identifier, and it cannot be the first character.

Scope A is nested within scope B if the identifier of scope A begins with the identifier of scope B followed by `\u0020`.

C>S:

```json
{
    "type": "command",
    "command": "list_scopes",
    "scope": null
}
```

If the `scope` argument is `null`, the server returns every scope in the simulation.

If not `null`, the `scope` argument allows the client to request scopes that are nested within that scope. Scopes from nested scopes are not returned. Large designs can contain many scopes, and limiting the amount of scopes loaded per request can afford a more responsive UI.

S>C:

```json
{
    "type": "response",
    "command": "list_scopes",
    "scopes": {
        "": {
            "type": "module",
            "definition": {
                "src": null,
                "name": null,
                "attributes": {}
            },
            "instantiation": {
                "src": null,
                "attributes": {}
            }
        },
        "soc": {
            "type": "module",
            "definition": {
                "src": "top.py:10",
                "name": "top.soc",
                "attributes": {
                    "top": {
                        "type": "unsigned_int",
                        "value": "1"
                    }
                }
            },
            "instantiation": {
                "src": "top.py:50",
                "attributes": {}
            }
        },
        "soc fifo": {
            "type": "module",
            "definition": {
                "src": "fifo.py:10",
                "name": "top.soc.fifo",
                "attributes": {}
            },
            "instantiation": {
                "src": "top.py:25",
                "attributes": {}
            }
        }
    }
}
```


#### Structure: *Scope Description*

A *Scope Description* is an object that has a name (its key in the scope dictionary), type, and type-specific information.

Currently, the only defined type is `"module"`:

```json
"soc": {
    "type": "module",
    "definition": {
        "name": "top.soc",
        "src": "top.py:10",
        "attributes": {
            "top": {
                "type": "unsigned_int",
                "value": "1"
            }
        }
    },
    "instantiation": {
        "src": "top.py:50",
        "attributes": {}
    }
}
```

For this scope type, separate information is available for the definition site and the instantiation site of the entity that was used to create the scope.

The value of `name` in a definition it is the name of the entity that was used to create this scope. It can be `null` if this information is somehow absent from the netlist, though this should be an uncommon case.

The value of `src` is exactly the same as the value of the corresponding `src` attribute; it is not validated or transformed. If the attribute is not present, the value is `null`.


#### Structure: *Attribute*

An *Attribute* is an object that has a name (its key in the attribute dictionary), type, and value:

```json
"src": {
    "type": "string",
    "value": "leds.py:10-13"
}
```

The possible types and the correspoinding value formats are:

* type `"unsigned_int"` is represented by a string matching a regular expression `[0-9]+`, which is the decimal representation of the integer value of the attribute;
* type `"signed_int"` is represented by a string matching a regular expression `[+-][0-9]+`, which is the decimal representation of the integer value of the attribute preceded by its sign;
* type `"string"` is represented by a string, which is the string value of the attribute;
* type `"double"` is represented by a floating-point number, which is the double precision value of the attribute.

```warning
The meaning of the attributes that are sent to the client is entirely netlist-specific. Attributes that are used for internal bookkeeping by the CXXRTL debug server (if any) are not sent to the client as a part of an attribute dictionary.
```


### Command: *List Items*

The *List Items* command allows the client to request the list of items and their descriptions. An item is a unit of design state. An item has an identifier, which is a string. There are two kinds of item identifiers:

* a string consisting of any characters except for `\u0020`: identifier of an item nested within the root scope.
* a scope identifier followed by `\u0020` followed by a string consisting of any characters except for `\u0020`: identifier of an item nested within a non-root scope.

Per the rules above, there can only be one `\u0020` character in a row in a scope identifier, and it cannot be the first character.

An item is nested within a scope if the identifier of the item begins with the identifier the scope followed by `\u0020`.

```note
A scope could contain both a nested scope and a nested item with the same name.
```

C>S:

```json
{
    "type": "command",
    "command": "list_items",
    "scope": null
}
```

If the `scope` argument is `null`, the server returns every item in the simulation.

If not `null`, the `scope` argument allows the client to request items that are nested within that scope. Items from nested scopes are not returned. Large designs can contain many thousands of items, and restricting loading of items to a scope while omitting items from nested scopes affords a more responsive UI.

S>C:

```json
{
    "type": "response",
    "command": "list_items",
    "items": {
        "clk": {
            "src": "top.py:15",
            "type": "node",
            "width": 1,
            "lsb_at": 0,
            "settable": true,
            "input": true,
            "output": false,
            "attributes": {
                "frequency": {
                    "type": "double",
                    "value": 1e6
                }
            }
        },
        "soc leds": {
            "src": "leds.py:10-13",
            "type": "node",
            "width": 16,
            "lsb_at": 0,
            "settable": false,
            "input": false,
            "output": false,
            "attributes": {}
        },
        "soc fifo mem": {
            "src": null,
            "type": "memory",
            "width": 32,
            "lsb_at": 0,
            "depth": 4096,
            "zero_at": 0,
            "settable": true,
            "attributes": {}
        }
    }
}
```

An item uniquely corresponds to exactly one identifier. The identifier does not change depending on the `scope` argument.


#### Structure: *Item Description*

An *Item Description* is an object that has a name (its key in the item dictionary), source location, type, and type-dependent properties.

The value of `src` is exactly the same as the value of the `src` attribute for the corresponding netlist entity; it is not validated or transformed. If the attribute is not present, the value is `null`.

The possible types are `"node"` and `"memory"`:

```json
"clk": {
    "src": "top.py:15",
    "type": "node",
    "lsb_at": 0,
    "width": 1,
    "input": true,
    "output": false,
    "settable": true,
    "attributes": {}
}
```

A *node* is a part of a netlist that has a one-dimensional value, representable as a single binary number. Primary inputs, primary outputs, outputs of combinational functions, outputs of storage registers, and some other parts of the netlist are nodes and have a corresponding *node item*.

A node may or may not be settable. Primary inputs and outputs of storage registers are settable, while primary outputs and outputs of combinational functions are not. In certain uncommon situations, only some of the bits of a node are settable. This is not reflected in the item description, which in this case describes the node as settable; the non-settable bits are ignored when the node's value is set.

```json
"soc fifo mem": {
    "src": null,
    "type": "memory",
    "lsb_at": 0,
    "width": 32,
    "zero_at": 0,
    "depth": 4096,
    "settable": true,
    "attributes": {}
}
```

A *memory* is a part of a netlist that has a two-dimensional value, representable as an array of binary numbers (each of which is called a *row*), which has a corresponding *memory item*. A memory contains `depth` rows.

A memory may or may not be settable, which is a property defined during synthesis. By default, all memories (even inferred read-only memories) are settable to permit initialization and patching of memories at the beginning of the simulation.

```note
At the moment the protocol does not include a command that sets the value of a node, which will be added at a later point.
```


### Command: *Reference Items*

The *Reference Items* command allows the client to associate a string identifier with a list of node items or memory item rows. This allows the client to retrieve 1d (for a time point) and 2d (for a time interval) arrays of item values without having to repeatedly send the (typically quite long, and rarely changing) item names. It enables other optimizations in the client and the server as well.

If, after an identifier is associated with items using the *Reference Items* command, another *Reference Items* command is sent with the same identifier, the identifier is reassociated and will point to the newly designated items. To free all server resources associated with an identifier that is no longer used, the `items` argument should be `null`.

If the identifier is an empty string, or an unknown item is designated, or memory rows are designated that are out of range of the memory depth, an error is returned.

C>S:

```json
{
    "type": "command",
    "command": "reference_items",
    "reference": "<reference>",
    "items": [
        ["soc leds"],
        ["soc fifo mem", 0, 3]
    ]
}
```

C>S:

```json
{
    "type": "command",
    "command": "reference_items",
    "reference": "<reference>",
    "items": null
}
```

S>C:

```json
{
    "type": "response",
    "command": "reference_items"
}
```


#### Structure: *Item Designation*

In the *Reference Items* command, an *Item Designation* refers to an item using one of two possible syntaxes depending on the item's type (as returned by the *List Items* command).
* A node item is designated in its entirety as `["<name>"]`.
* Memory item rows are designated with a range `["<name>", first, last]` where `first` and `last` are integers that are the indices of the first and last designated row, inclusive. (In total, `last - first + 1` rows will be designated.) Rows could be designated in either ascending or descending direction.

If an item is designated using a syntax not matching its type, an error is returned.


### Command: *Query Interval*

The *Query Interval* command allows the client to retrieve the contents of the waveform database for a given time interval. The waveform database stores a time series of samples: for an ordered (non-descending) set of time points, each has associated item values and diagnostics. Thus, the *Query Interval* command selects a subset of time points falling within the given range, and for each time point, it returns the requested data (item values, unless `items` is `null`, and diagnostics, unless `diagnostics` is `false`).

```note
The overhead of running the *Query Interval* command twice (once to retrieve the item values and once to retrieve the diagnostics) varies depending on the exact contents of the waveform database. In case both are needed, it is always more efficient to run the command once while requesting both.
```

Both the begin and end of the time interval must be between `0.0` and the latest stored time point (as may be retrieved using the *Get Simulation Status* command). If this is not the case an error is returned.

If the `items` argument does not designate a non-empty list of items, an error is returned.

C>S:

```json
{
    "type": "command",
    "command": "query_interval",
    "interval": ["0.0", "0.100000000000000"],
    "collapse": true,
    "items": "<reference>",
    "item_values_encoding": "base64(u32)",
    "diagnostics": true
}
```

The `item_values_encoding` argument determines the encoding of the `item_values` field in the returned samples. The `base64(u32)` encoding is always supported. If this argument is specified as `null`, no item values are returned (the behavior is the same as when the `items` argument is specified as `null`).

S>C:

```json
{
    "type": "response",
    "command": "query_interval",
    "samples": [
        {
            "time": "0.000000000000000",
            "item_values": "AAAAAA==",
            "diagnostics": [],
        },
        {
            "time": "0.100000000000000",
            "item_values": "AAAAAQ==",
            "diagnostics": [
                {
                    "type": "assert",
                    "text": "Assertion (x == 1) failed",
                    "src": "/home/user/design/fifo.py:130|/home/user/design/top.py:10"
                }
            ]
        }
    ]
}
```

It is possible to request samples without either item values or diagnostics. This is useful to find out at which time points does the database contain samples.

C>S:

```json
{
    "type": "command",
    "command": "query_interval",
    "interval": ["0.0", "0.100000000000000"],
    "collapse": true,
    "items": null,
    "item_values_encoding": null,
    "diagnostics": false
}
```

S>C:

```json
{
    "type": "response",
    "command": "query_interval",
    "samples": [
        {
            "time": "0.000000000000000",
        },
        {
            "time": "0.100000000000000",
        }
    ]
}
```

Samples are returned in non-descending time order, and the `time` of each returned sample falls within the `range` argument of the command, with the exception of the first sample, whose `time` may be less than the beginning of the range.

```note
The intent of the *Query Interval* command is to return information for the entirety of the requested range. If only a single time point is of interest (i.e. the begin and end of the time interval are the same), and the waveform database contains a sample before that time point, this command will return that sample even though it does not fall within the range.

An alternative to this would be to clamp the returned `time` to the requested `range`, but this complicates presentation (making it more difficult to retrieve contents of successive time intervals, e.g. during scrolling), interferes with caching, and misleads the designer, who typically expects the simulation to run with a specific fixed resolution of which all time points are a multiple of.
```

If collapsed samples are requested, there will be only one sample for each time point. This mode is useful for waveform display and general design interaction.

If non-collapsed samples are requested, there may be multiple samples that have the same time point, but differing item values and messages. This mode is useful for obtaining a dump of the replay buffer and for debugging race conditions in zero time. In most circumstances, it should not be necessary to use this mode, and collapsed samples should be requested to reduce overhead of data encoding and transfer.

```warning
Because the simulation can progress without advancement of time, if the end of the requested time interval is the same as the latest time point stored in the waveform database, two subsequent invocations of the *Query Interval* command that requests collapsed samples can have the last sample differ between them. Be mindful of this edge case when displaying samples from a running simulation.
```


#### Structure: *Time Point*

A *Time Point* is a string matching the regular expression `^\d+\.\d+$`. It consists of two decimal integers delimited by a U+002E character. The two parts, combined together, specify absolute time as follows:

* The first part specifies the number of whole seconds since the beginning of time. Its value may not exceed 2147483647 (2 to the power of 31, minus one).
* The second part specifies the number of whole femtoseconds since the end of the last whole second. Its value may not exceed 999999999999999 (10 to the power of 15, minus one).


#### Structure: *Time Interval*

A *Time Interval* is an array `[begin, end]` where both `begin` and `end` is a *Time Point*. The range is inclusive of both the begin and end points.


#### Structure: *Item Values* (`base64(u32)`)

*Item Values* in the `base64(u32)` format are represented as a sequence of 32-bit little-endian words encoded using [Base64](https://datatracker.ietf.org/doc/html/rfc4648#section-4), with padding included. The sequence of the words is constructed by concatenating, for each of the items in the order in which they were designated using the *Reference Items* command, their binary representation, from the least to the most significant 32-bit word.

```note
CXXRTL internally stores item values as sequences of 32-bit words, from least to most significant, using the fewest amount of words sufficient to represent the value. This encoding is not particularly space-efficient (especially for designs with many 1-bit signals; it has an overhead of 32× for fine netlists), but is very uniform. The uniformity allows aggregating item values into arrays of 32-bit integers and storing, for each item, only the offset into the array.
```

```note
Because both the bytes within a word and the words themselves are stored in little-endian form, each individual item value can be treated as a single number, stored in little-endian form, and padded up to the next 4-byte boundary.
```


#### Structure: *Diagnostic*

A *Diagnostic* is an object that has a type, text, and source location:

```json
{
    "type": "assert",
    "text": "Assertion (x == 1) failed",
    "src": "/home/user/design/fifo.py:130|/home/user/design/top.py:10"
}
```

The possible diagnostic types are `"break"`, `"print"`, `"assert"`, `"assume"`.

The diagnostic text is a string containing arbitrary text that may include embedded newlines, and may or may not be newline-terminated.

```note
Diagnostics of type `"print"` may be emitted in batches, where only the last diagnostic is guaranteed to be newline-terminated. To be displayed in a manner that is expected by the designer, the client should treat such diagnostics as fragments of text written to a line-buffered stream: instead of immediately displaying each such diagnostic on its own line, the client should reassemble the fragments in a buffer, and display each of the complete lines as they appear in the buffer.

Diagnostics of type `"break"`, `"assert"`, `"assume"` are self-contained, and their `text` does not include a terminating newline.
```

The value of `src` is exactly the same as the value of the `src` attribute for the corresponding netlist entity; it is not validated or transformed. If the attribute is not present, the value is `null`.


#### Structure: *Sample*

A *Sample* is an object that has a time, item values (if requested), and diagnostics (if requested):

```json
{
    "time": "0.000000000000000",
    "item_values": "AAAAAA==",
    "diagnostics": [],
}
```

If either item values or diagnostics are not requested the corresponding field will be absent. For example, using `"diagnostics": false` in the *Query Interval* command above will yield samples with the following format:

```json
{
    "time": "0.000000000000000",
    "item_values": "AAAAAA=="
}
```


### Command: *Get Simulation Status*

The *Get Simulation Status* command queries whether the simulation is running and returns the latest time point that has been stored to the waveform database.

```note
If the simulation status is `"running"`, then by the time the response is recevied and processed by the client, additional samples may have been added to the waveform database, advancing the latest time point.
```

C>S:

```json
{
    "type": "command",
    "command": "get_simulation_status"
}
```

S>C:

```json
{
    "type": "response",
    "command": "get_simulation_status",
    "status": "running",
    "latest_time": "0.125000000000000"
}
```

S>C:

```json
{
    "type": "response",
    "command": "get_simulation_status",
    "status": "paused",
    "latest_time": "0.125000000000000",
    "next_sample_time": "0.126000000000000"
}
```

The possible simulation statuses are `"running"`, `"paused"`, and `"finished"`. If the status is `"paused"`, an additional field, `next_sample_time`, indicates the time point of the next sample that will be taken when the simulation is resumed.

There is always at least one initial sample, at the zero time, in the waveform database. Therefore, `latest_time` is either `"0.0"` or a greater time point.


### Command: *Run Simulation*

The *Run Simulation* command starts the simulation, asynchronously to command processing. The simulation will continue to run until one of the following events occurs:

* The simulation time advances to or past the time specified in the `until_time` argument (if it is not `null`). A *Simulation Paused* event with `cause` of `"until_time"` is sent to the client.
* A diagnostic with one of the types specified in the `until_diagnostics` argument is emitted. A *Simulation Paused* event with the `cause` of `"until_diagnostics"` is sent to the client.
* The simulation is paused by the client with the *Pause Simulation* command. No event is sent.

```note
It is possible for the simulation to be paused after this command is issued when `latest_time` is less than `until_time`, if the simulation time advances past `until_time`. In this case, no samples at time points beyond `until_time` are added to the simulation database, and the simulation time is returned by the *Get Simulation Status* command in the `next_sample_time` argument.
```

If the simulation status is not `"paused"` an error is returned.

C>S:

```json
{
    "type": "command",
    "command": "run_simulation",
    "until_time": null,
    "until_diagnostics": [],
    "sample_item_values": true
}
```

The `sample_item_values` argument controls whether item values are written to the waveform database. If it is `false`, samples in the time interval from the latest time point when the *Run Simulation* command is sent and until the simulation stops will not have item values (the *Query Interval* command will return `null` in the `item_values` field).

```note
Essential sample data is still written to the waveform database at the beginning and the end of the time interval that has the sampling disabled, but the overhead of writing this data is minimal.

Diagnostics are always written to the waveform database.
```

S>C:

```json
{
    "type": "response",
    "command": "run_simulation"
}
```


### Command: *Pause Simulation*

The *Pause Simulation* command synchronously stops the simulation. It does not cause a *Simulation Paused* event to be sent.

If the simulation status is `"paused"` or `"finished"` this command does nothing.

C>S:

```json
{
    "type": "command",
    "command": "pause_simulation"
}
```

S>C:

```json
{
    "type": "response",
    "command": "pause_simulation",
    "time": "0.300000000000000"
}
```


## Events

*Events* are asynchronous notifications of changes in the design state or of actions performed by the asynchronously executed simulation.


### Event: *Simulation Paused*

The *Simulation Paused* event informs the client that the status of the simulation has asynchronously (without the client explicitly requesting it) changed to `"paused"`.

S>C:

```json
{
    "type": "event",
    "event": "simulation_paused",
    "time": "0.300000000000000",
    "cause": "until_time"
}
```

The `time` argument is the time at which the simulation has paused, which is also the time of the latest time point that has been stored in the simulation database. It is equal to the `latest_time` returned by a *Get Simulation Status* command issued afterwards.

The list of possible causes is:

* `"until_time"`
* `"until_diagnostics"`


### Event: *Simulation Finished*

The *Simulation Finished* event informs the client that the status of the simulation has changed to `"finished"`.

S>C:

```json
{
    "type": "event",
    "event": "simulation_finished",
    "time": "0.300000000000000"
}
```

The `time` argument is the time of the final time point that has been stored in the simulation database. It is equal to the `latest_time` returned by a *Get Simulation Status* command issued afterwards.


## Initial state

Right after the CXXRTL debug server is started, the simulation contains one initial sample at the time point `"0.0"`. No communication with the server is possible until it has performed all of the necessary initialization, evaluated the initial state, and stored a sample of that state in the simulation database.

