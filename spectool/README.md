# JACDAC specification tool

The jdspectool takes a service specification in markdown format and converts it to JSON.

It can then also generate template files for implementation in various languages.

## jdspectool format

The tool uses literate programming in markdown format.
The formal specification fragments, given as markdown code sections, are interleaved with
English specification text.
The code sections can be in triple backticks or with the 4 leading spaces.
Examples in this document only use the leading spaces form.

The bulk of the specification talks about packet formats.
There are three main types of packets described: registers, commands, and events.

## Example

First, let's go through a simple example, commenting on various functionality.

```markdown
# Rocket engine

    identifier: 0x17d50e4b

A controller for a liquid-propelled rocket.
```

The first line specifies the name of the service (`name` in JSON format).

Then we assign is unique identifier (`classIdentifier` in JSON format).
These identifers are always of the form `0x1-------`.
If you're just defining a new service, specify it as `1` or other invalid value
and the jdspectool will suggest a valid random identifier.

What follows is a short description of the service (`notes["short"]` in JSON).

```markdown
## History

The idea of liquid rocket first appears in a book by Konstantin Tsiolkovsky.
This type of rockets carried humans to moon in Apollo 11.
```

Further non-normative sections can follow, which are stored in `notes["long"]` in JSON.

```markdown
## Registers

All register writes incur 1.25ms delay.

    rw oxygen_delay = 100: u16 ms @ 0x80

The time to wait after starting hydrogen flow, before starting oxygen flow.

    ro core_temperature: u32 K @ 0x180

The temperature at engine core.

    const num_nozzles: u8 @ 0x181

The number of exhaust nozzles in this rocket.

    ro acceleration @ 0x182 {
        x: i22.10 g
        y: i22.10 g
        z: i22.10 g
    }

Current acceleration forces experienced by the rocket. Includes gravity.
```

The first line of registers section goes in `notes["registers"]` in JSON,
and is meant to be displayed along with every register description in various UIs.

The first register we define can be read and written.
When the device restarts, its value is `100` (default values are
typically `0` and in that case are not specified).
The value is unsigned and 16 bit long, and is expressed in milliseconds.
The register code is `0x80` (code ranges are specified in `_base.md`).
The text after the register definition ends up in `description` field in JSON.
Register name (nor description) is never sent over wire (same applies to commands
and events).

The next line specifies a read-only register, giving current temperature in degrees Kelvin.
A 32 bit value is used to avoid overflows.

The third register is unit-less.
It is also one that does not change, at least until the device resets
(in the case of this service, the reset is usually quite spectacular).

The fourth register is current acceleration.
The `i22.10` type is signed 32 bit integer, where the value is shifted by 10 bits.
That is, the last 10 bits is the fractional part.
The values are expressed in `g`, which is Earth-standard gravity.

The register also has three fields, instead of the usual one.
In fact, the `core_temperature` could be also expressed using the more verbose syntax,
using `_` as the field name:

```
ro core_temperature @ 0x180 {
    _: u32 K
}
```

Now, let's move on to commands.

```markdown
## Commands

    command launch @ 0x80 {
        launch_secret_code: u32
        delay: u32 ms
    }
    report {
        launch_id: u32;
    }

Fire up the rocket after `delay` ms. A correct launch code has to be provided.

    command abort @ 0x81 {
        launch_id: u32
    }

Cancel previously scheduled launch.

    command self_destroy @ 0x82 { }

Boom!
```

Here, no `note["commands"]` is provided (if it were, it would be used similar to `note["registers"]`).
The first command is used to initiate launch sequence.
The controller will respond with a number identifier for this particular launch, which
can be used to abort it with the second command.
Note that `report` syntax skips name and command code - these are copied from the preceding
`command`.

Also note that `0x80` command code is used, which is the same as the register code for 
`oxygen_delay`.
This is allowed, as register codes occupy separate namespace from command codes -
register codes combined with masks of `0x1000` or `0x2000` form command codes
for getting or setting registers.
Again, `_base.md` has ranges for allowed command codes.

The next command doesn't have a associated report.
The sender can still ask for ACK.
It could have been equivalently specified with the short form as:

```
command abort: u32 @ 0x81
```

However, that would miss the informative field name `launch_id`.

Finally, some commands don't need arguments, like the `self_destroy` one.
The empty braces `{ }` can also be skipped.

Next, let's move on to events.

```markdown
## Events

All events are emitted three times, to ensure good reception.

    event take_off @ 0x01

Rockets leaves the launch pad.

    event bird_in_nozzle @ 0x02 {
        nozzle_id: u8
    }

Indicates that nozzle is about to fail because of a forign object.
```

First, there's a `note["events"]`.
The strategy mentioned is valid, though this is better achieved with streams.

The address space for events is separate from commands and registers, as they are nested
inside of the `event` command.
This 32 bit are available, it's recommended to stick to low numbers.

Many events have no arguments, like the `take_off` one.
Arguments can be specified as for registers and commands.

## Extending specs

Service specs can extend certain base specs.
Currently, this mostly applies to `_sensor.md` spec.
For example:

```markdown
# Button

    identifier: 0x1473a263
    extends: _sensor

A simple push-button.

## Registers

    ro pressed: bool @ reading

Indicates whether the button is currently active (pressed).

## Events

    event down @ 0x01

Emitted when button goes from inactive (`pressed == 0`) to active.

    event up @ 0x02

Emitted when button goes from active (`pressed == 1`) to inactive.
```

All registers from the `_sensor.md` will be included (meaning `is_streaming` and `streaming_interval`).
Similarly, all commands and events would be included (but there are none).

The address of `pressed` register is specified as `@ reading`, meaning the
address of the `reading` register in `_base.md`.
Services cannot extend `_base.md`, because it includes a large number of packets that are
never all present at the same time in a regular service.
However, the names of these packets can be used in places of addresses.
In fact, this is the only way to refer to the addresses in the system range:
if the example used `@ 0x101` instead of `@ reading`, the jdspectool would
generate an error.

## Other features

jdspectool validates ranges of commands and register addresses.
If you want to use high commands or registers (above 0x100/0x200; note that for vast majority
of services this should not be needed),
include `high: 1` after `identifier: ...` (the error message will tell you that).

## Planned features

```
register temperature: i8 C @ reading [
    typical_min: -20,
    typical_max: 40
] 
```
