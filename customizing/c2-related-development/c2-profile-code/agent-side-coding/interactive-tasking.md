# Interactive Tasking

## Message Structure

Messages for interactive tasking have three pieces:

```
{
    "task_id": "UUID of task",
    "data": "base64 of data",
    "message_type": int enum of types
}
```

If you have a command called `pty` and issue it, then when that task gets sent to your agent, you have your normal tasking structure. That tasking structure includes an id for the task that's a UUID. All follow-on interactive input for that task uses the same UUID (`task_id` in the above message).

The `data` is pretty straight forward - it's the base64 of the raw data you're trying to send to/from this interactive task. The `message_type` field is an enum of `int`.  It might see complicated at first, but really it boils down to providing a way to support sending control codes through the web UI, scripting, and through an opened port.

{% code lineNumbers="true" fullWidth="false" %}
```go
const (
   Input = 0
   Output = 1
   Error = 2
   Exit = 3
   Escape = 4    //^[ 0x1B
   CtrlA = 5     //^A - 0x01 - start
   CtrlB = 6     //^B - 0x02 - back
   CtrlC = 7     //^C - 0x03 - interrupt process
   CtrlD = 8     //^D - 0x04 - delete (exit if nothing sitting on input)
   CtrlE = 9     //^E - 0x05 - end
   CtrlF = 10     //^F - 0x06 - forward
   CtrlG = 11     //^G - 0x07 - cancel search
   Backspace = 12 //^H - 0x08 - backspace
   Tab = 13       //^I - 0x09 - tab
   CtrlK = 14     //^K - 0x0B - kill line forwards
   CtrlL = 15     //^L - 0x0C - clear screen
   CtrlN = 16     //^N - 0x0E - next history
   CtrlP = 17     //^P - 0x10 - previous history
   CtrlQ = 18     //^Q - 0x11 - unpause output
   CtrlR = 19     //^R - 0x12 - search history
   CtrlS = 20     //^S - 0x13 - pause output
   CtrlU = 21     //^U - 0x15 - kill line backwards
   CtrlW = 22     //^W - 0x17 - kill word backwards
   CtrlY = 23     //^Y - 0x19 - yank
   CtrlZ = 24     //^Z - 0x1A - suspend process
   end
)
```
{% endcode %}

When something is coming from Mythic -> Agent, you'll typically see `Input`, `Exit`, or `Escape` -> `CtrlZ`. When sending data back from Agent -> Mythic, you'll set either `Output` or `Error`. This enum example also includes what the user typically sees in a terminal (ex: `^C` when you type CtrlC) along with the hex value that's normally sent. Having data split out this way can be helpful depending on what you're trying to do. Consider the case of trying to do a `tab-complete`. You want to send down data _and_ the tab character (in that order). For other things though, like `escape`, you might want to send down `escape` and then data (in that order for things like control sequences).&#x20;

You'll probably notice that some letters are missing from the control codes above. There's no need to send along a special control code for `\n` or `\r` because we can send those down as part of our input. Similarly, clearing the screen isn't useful through the web UI because it doesn't quite match up as a full TTY.



## Message Location

This data is located in a similar way to SOCKS and RPFWD:

```
{
    "action": "some action",
    "interactive": [ {"task_id": UUID, "data": "base64", "message_type": 0 } ]
}
```

the `interactive` keyword takes an array of these sorts of messages to/from the agent. This keyword is at the same level in the JSON structure as `action`, `socks`, `responses`, etc.
