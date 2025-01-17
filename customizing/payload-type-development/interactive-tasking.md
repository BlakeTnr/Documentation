# Interactive Tasking

Using a C2 platform, you don't generally have the ability to send follow-on information to a running task. Each task is run in isolation, especially if you're running a "shell" command. However, there are times when it's very useful to have an interactive shell on the target host. Depending on the operating system and configuration, you could of course attempt to use SOCKS to make a local SSH connection, or even use RPFWD to cause a shell to make a "remote" connection to yourself and tunnel it externally. These have their own pros/cons.

Another option would be to allow follow-on input within your current task without requiring additional network connections or going to something like `sleep 0` for it to work. This is where "interactive" tasking comes into play.&#x20;

If your command specifies a `supported_ui_feature` of `task_response:interactive`, then Mythic will display a slightly different interface for that task's output.&#x20;

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

Notice here that as part of the task response, there's a whole new set of controls and input fields. The controls allow you to send things like CtrlC, CtrlZ, CtrlD through the browser down to your agent. Additionally, on the right-hand side there's a control for what kind of line endings to set when you hit enter - None, LF, CR, or CRLF. The `None` option is particularly useful you want to send something along like "q" or " " to the other side without sending "q\n" or " \n".&#x20;

The other three buttons there allow you to toggle ANSI Terminal coloring, indicators for your tasking, and word wrapping respectively.&#x20;

Just like with SOCKS and RPFWD, you can use MythicRPCProxyStart to open up a port with this sort of task. If you do, then you can interactively work with the task through your own terminal instead of through the web UI.&#x20;

**Note:** When tasking through the UI, all input is tracked as normal tasking, but no "tasks" are created when interacting via the opened port. Output is still saved and displayed within the UI.

For information on the message format, check out [Interactive Tasking](../c2-related-development/c2-profile-code/agent-side-coding/interactive-tasking.md).
