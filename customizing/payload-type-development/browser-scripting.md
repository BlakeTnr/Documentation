# Browser Scripting

## What is Browser Scripting?

Browser scripting is a way for you, the agent developer or the operator, to dynamically adjust the output that an agent reports back. You can turn data into tables, download buttons, screenshot viewers, and even buttons for additional tasking.

## Where are Browser Scripts?

As a developer, your browser scripts live in a folder called `browser_scripts` in your `mythic` folder. These are simply JavaScript files that you then reference from within your command files such as:

```python
browser_script = BrowserScript(script_name="download_new", author="@its_a_feature_", for_new_ui=True)
```

As an operator, they exist in the web interface under "Operations" -> "Browser Scripts". You can enable and disable these for yourself at any time, and you can even modify or create new ones from the web interface as well. If you want these changes to be persistent or available for a new Mythic instance, you need to save it off to disk and reference it via the above method.

{% hint style="info" %}
You need to supply `for_new_ui=True` in order for the script to be leveraged for the new user interface. If you don't do this, Mythic will attach the script to the old user interface. All of the following documentation is for the new user interface.
{% endhint %}

## What can I do via Browser Scripts?

Browser Scripts are JavaScript files that take in a reference to the task and an array of the responses available, then returns a Dictionary representing what you'd like Mythic to render on your behalf. This is pretty easy if your agent returns structured output that you can then parse and process. If you return unstructured output, you can still manipulate it, but it will just be harder for you.

### Plaintext

The most basic thing you can do is return plaintext for Mythic to display for you. Let's take an example that simply aggregates all of the response data and asks Mythic to display it:

```javascript
function(task, responses){
    const combined = responses.reduce( (prev, cur) => {
            return prev + cur;
        }, "");
    return {'plaintext': combined};
}
```

This function reduces the Array called `responses` and aggregates all of the responses into one string called `combined` then asks Mythic to render it via: `{'plaintext': combined}`.

### Screenshots

A slightly more complex example is to render a button for Mythic to display a screenshot.

```javascript
function(task, responses){
    if(task.status.toLowercase().includes("error")){
        const combined = responses.reduce( (prev, cur) => {
            return prev + cur;
        }, "");
        return {'plaintext': combined};
    }else if(task.completed){
        if(responses.length > 0){
            let data = JSON.parse(responses[0]);
            return {"screenshot":[{
                "agent_file_id": data["agent_file_id"],
                "variant": "contained",
                "name": "View Screenshot"
            }]};
        }else{
            return {"plaintext": "No data to display..."}
        }

    }else if(task.status === "processed"){
        // this means we're still downloading
        if(responses.length > 0){
            let data = JSON.parse(responses[0]);
            return {"screenshot":[{
                "agent_file_id": data["agent_file_id"],
                "variant": "contained",
                "name": "View Partial Screenshot"
            }]};
        }
        return {"plaintext": "No data yet..."}
    }else{
        // this means we shouldn't have any output
        return {"plaintext": "Not response yet from agent..."}
    }
}
```

This function does a few things:

1. If the task status includes the word "error", then we don't want to process the response like our standard structured output because we returned some sort of error instead. In this case, we'll do the same thing we did in the first step and simply return all of the output as `plaintext`.
2. If the task is completed and isn't an error, then we can verify that we have our responses that we expect. In this case, we simply expect a single response with some of our data in it. The one piece of information that the browser script needs to render a screenshot is the `agent_file_id` or `file_id` of the screenshot you're trying to render. If you want to return this information from the agent, then this will be the same `file_id` that Mythic returns to you for transferring the file. If you display this information via `process_response` output from your agent, then you're likely to pull the file data via an RPC call, and in that case, you're looking for the `agent_file_id` value.
3. To actually create a screenshot, we return a dictionary with a key called `screenshot` that has an array of Dictionaries. We do this so that you can actually render multiple screenshots at once (such as if you fetched information for multiple monitors at a time). For each screenshot, you just need three pieces of information: the `agent_file_id`, the `name` of the button you want to render, and the `variant` is how you want the button presented (`contained` is a solid button and `outlined` is just an outline for the button).
4. If we didn't error and we're not done, then the status will be `processed`. In that case, if we have data we want to also display the partial screenshot, but if we have no responses yet, then we want to just inform the user that we don't have anything yet.

### Tables

Creating tables is a little more complicated, but not by much. The biggest thing to consider is that you're asking Mythic to create a table for you, so there's a few pieces of information that Mythic needs such as what are the headers, are there any custom styles you want to apply to the rows or specific cells, what are the rows and what's the column value per row, in each cell do you want to display data, a button, issue more tasking, etc. So, while it might seem overwhelming at first, it's really nothing too crazy.

Let's take an example and then work through it - we're going to render the following screenshot

![File Listing Browser Script](<../../.gitbook/assets/Screen Shot 2021-10-10 at 5.10.33 PM.png>)

```javascript
function(task, responses){
    if(task.status.includes("error")){
        const combined = responses.reduce( (prev, cur) => {
            return prev + cur;
        }, "");
        return {'plaintext': combined};
    }else if(task.completed && responses.length > 0){
        let folder = {
                    backgroundColor: "mediumpurple",
                    color: "white"
                };
        let file = {};
        let data = "";
        try{
            data = JSON.parse(responses[0]);
        }catch(error){
           const combined = responses.reduce( (prev, cur) => {
                return prev + cur;
            }, "");
            return {'plaintext': combined};
        }
        let ls_path = "";
        if(data["parent_path"] === "/"){
            ls_path = data["parent_path"] + data["name"];
        }else{
            ls_path = data["parent_path"] + "/" + data["name"];
        }
        let headers = [
            {"plaintext": "name", "type": "string"},
            {"plaintext": "size", "type": "size"},
            {"plaintext": "owner", "type": "string"},
            {"plaintext": "group", "type": "string"},
            {"plaintext": "posix", "type": "string", "width": 8},
            {"plaintext": "xattr", "type": "button", "width": 10},
            {"plaintext": "DL", "type": "button", "width": 6},
            {"plaintext": "LS", "type": "button", "width": 6},
            {"plaintext": "CS", "type": "button", "width": 6}
        ];
        let rows = [{
            "rowStyle": data["is_file"] ? file : folder,
            "name": {"plaintext": data["name"]},
            "size": {"plaintext": data["size"]},
            "owner": {"plaintext": data["permissions"]["owner"]},
            "group": {"plaintext": data["permissions"]["group"]},
            "posix": {"plaintext": data["permissions"]["posix"]},
            "xattr": {"button": {
                "name": "View XATTRs",
                "type": "dictionary",
                "value": data["permissions"],
                "leftColumnTitle": "XATTR",
                "rightColumnTitle": "Values",
                "title": "Viewing XATTRs"
            }},
            "DL": {"button": {
              "name": "DL",
              "type": "task",
              "disabled": !data["is_file"],
              "ui_feature": "file_browser:download",
              "parameters": ls_path
            }},
            "LS": {"button": {
                "name": "LS",
                "type": "task",
                "ui_feature": "file_browser:list",
                "parameters": ls_path
            }},
            "CS": {"button": {
                "name": "CS",
                "type": "task",
                "ui_feature": "code_signatures:list",
                "parameters": ls_path
                }}
        }];
        for(let i = 0; i < data["files"].length; i++){
            let ls_path = "";
            if(data["parent_path"] === "/"){
                ls_path = data["parent_path"] + data["name"] + "/" + data["files"][i]["name"];
            }else{
                ls_path = data["parent_path"] + "/" + data["name"] + "/" + data["files"][i]["name"];
            }
            let row = {
                "rowStyle": data["files"][i]["is_file"] ? file:  folder,
                "name": {"plaintext": data["files"][i]["name"]},
                "size": {"plaintext": data["files"][i]["size"]},
                "owner": {"plaintext": data["files"][i]["permissions"]["owner"]},
                "group": {"plaintext": data["files"][i]["permissions"]["group"]},
                "posix": {"plaintext": data["files"][i]["permissions"]["posix"],
                    "cellStyle": {

                    }
                },
                "xattr": {"button": {
                    "name": "View XATTRs",
                    "type": "dictionary",
                    "value": data["files"][i]["permissions"],
                    "leftColumnTitle": "XATTR",
                    "rightColumnTitle": "Values",
                    "title": "Viewing XATTRs"
                }},
                "DL": {"button": {
                  "name": "DL",
                  "type": "task",
                    "disabled": !data["files"][i]["is_file"],
                  "ui_feature": "file_browser:download",
                  "parameters": ls_path
                }},
                "LS": {"button": {
                    "name": "LS",
                    "type": "task",
                    "ui_feature": "file_browser:list",
                    "parameters": ls_path
                }},
                "CS": {"button": {
                    "name": "CS",
                    "type": "task",
                    "ui_feature": "code_signatures:list",
                    "parameters": ls_path
                }}
            };
            rows.push(row);
        }
        return {"table":[{
            "headers": headers,
            "rows": rows,
            "title": "File Listing Data"
        }]};
    }else if(task.status === "processed"){
        // this means we're still downloading
        return {"plaintext": "Only have partial data so far..."}
    }else{
        // this means we shouldn't have any output
        return {"plaintext": "Not response yet from agent..."}
    }
}
```

This looks like a lot, but it's nothing crazy - there's just a bunch of error handling and dealing with parsing errors or task errors. Let's break this down into a few easier to digest pieces:

```javascript
return {"table":[{
            "headers": headers,
            "rows": rows,
            "title": "File Listing Data"
        }]};
```

In the end, we're returning a dictionary with the key `table` which has an array of Dictionaries. This means that you can have multiple tables if you want. For each one, we need three things: information about headers, the rows, and the title of the table itself. Not too bad right? Let's dive into the headers:

```javascript
let headers = [
            {"plaintext": "name", "type": "string"},
            {"plaintext": "size", "type": "size"},
            {"plaintext": "owner", "type": "string"},
            {"plaintext": "group", "type": "string"},
            {"plaintext": "posix", "type": "string", "width": 8},
            {"plaintext": "xattr", "type": "button", "width": 10},
            {"plaintext": "DL", "type": "button", "width": 6},
            {"plaintext": "LS", "type": "button", "width": 6},
            {"plaintext": "CS", "type": "button", "width": 6}
        ];
```

Headers is an array of Dictionaries with three values each - `plaintext`, `type`, and optionally `width`. As you might expect, `plaintext` is the value that we'll actually use for the title of the column. `type` is controlling what kind of data will be displayed in that column's cells. There are a few options here: `string` (just displays a standard string), `size` (takes a size in bytes and converts it into something human readable - i.e. 1024 -> 1KB), `date` (process date values and display them and sort them properly), `number` (display numbers and sort them properly), and finally `button` (display a button of some form that does something). The last value here is `width` - this is a percentage of how much width you want the column to take up by default. If you don't specify anything here, then all columns take up an equal amount of width. In the above example, the first four columns will be equal in size, then `posix` will start out at 8%, `xattr` at 10%, etc.&#x20;

Now let's look at the actual rows to display:

```javascript
        let rows = [{
            "rowStyle": data["is_file"] ? file : folder,
            "name": {"plaintext": data["name"]},
            "size": {"plaintext": data["size"]},
            "owner": {"plaintext": data["permissions"]["owner"]},
            "group": {"plaintext": data["permissions"]["group"]},
            "posix": {"plaintext": data["permissions"]["posix"]},
            "xattr": {"button": {
                "name": "View XATTRs",
                "type": "dictionary",
                "value": data["permissions"],
                "leftColumnTitle": "XATTR",
                "rightColumnTitle": "Values",
                "title": "Viewing XATTRs"
            }},
            "DL": {"button": {
              "name": "DL",
              "type": "task",
              "disabled": !data["is_file"],
              "ui_feature": "file_browser:download",
              "parameters": ls_path
            }},
            "LS": {"button": {
                "name": "LS",
                "type": "task",
                "ui_feature": "file_browser:list",
                "parameters": ls_path
            }},
            "CS": {"button": {
                "name": "CS",
                "type": "task",
                "ui_feature": "code_signatures:list",
                "parameters": ls_path
                }}
        }];
```

Ok, lots of things going on here, so let's break it down:

#### rowStyle

As you might expect, you can use this key to specify custom styles for the row overall. In this example, we're adjusting the display based on if the current row is for a file or a folder.

#### plaintext

If we're displaying anything other than a button for a column, then we need to include the `plaintext` key with the value we're going to use. You'll notice that aside from `rowStyle`, each of these other keys match up with the `plaintext` header values so that we know which values go in which columns.

#### dictionary button

The first kind of button we can do is just a popup to display additional information that doesn't fit within the table. In this example, we're displaying all of Apple's extended attributes via an additional popup.&#x20;

```javascript
{"button": {
                "name": "View XATTRs",
                "type": "dictionary",
                "value": data["permissions"],
                "leftColumnTitle": "XATTR",
                "rightColumnTitle": "Values",
                "title": "Viewing XATTRs"
            }}
```

The button field takes a few values, but nothing crazy. `name` is the name of the button you want to display to the user. the `type` field is what kind of button we're going to display - in this case we use `dictionary` to indicate that we're going to display a dictionary of information to the user. The other type is `task` that we'll cover next. The `value` here should be a Dictionary value that we want to display. We'll display the dictionary as a table where the first column is the key and the second column is the value, so we can provide the column titles we want to use. We can optionally make this button disabled by providing a `disabled` field with a value of `true`.  Lastly, we provide a `title` field for what we want to title the overall popup for the user.

#### task button

This button type allows you to issue additional tasking.&#x20;

```javascript
{"button": {
              "name": "DL",
              "type": "task",
              "disabled": !data["is_file"],
              "ui_feature": "file_browser:download",
              "parameters": ls_path
            }}
```

This button has the same `name` and `type` fields as the dictionary button. Just like with the dictionary button we can make the button disabled or not with the `disabled` field. You might be wondering which task we'll invoke with the button. This works the same way we identify which command to issue via the file browser or the process browser - `ui_feature`. These can be anything you want, just make sure you have the corresponding feature listed somewhere in your commands or you'll never be able to task it.

The last thing here is the `parameters`. If you provide parameters, then Mythic will automatically use them when tasking. In this example, we're pre-creating the full path for the files in question and passing that along as the parameters to the `download` function. If you don't provide any parameters and the task you're trying to issue takes parameters, then you will get a popup to provide the parameters, just like if you tasked it from the command line.

#### menu button

Tasking and extra data display button is nice and all, but if you have a lot of options, you don't want to have to waste all that valuable text space with buttons. To help with that, there's one more type of button we can do: `menu`. With this we can wrap the other kinds of buttons:

```json
"button": {
    "name": "Actions",
    "type": "menu",
    "value": [
            {
                "name": "View XATTRs",
                "type": "dictionary",
                "value": data["files"][i]["permissions"],
                "leftColumnTitle": "XATTR",
                "rightColumnTitle": "Values",
                "title": "Viewing XATTRs"
            },
            {
                "name": "Get Code Signatures",
                "type": "task",
                "ui_feature": "code_signatures:list",
                "parameters": ls_path
            },
            {
                "name": "LS Path",
                "type": "task",
                "ui_feature": "file_browser:list",
                "parameters": ls_path
            },
            {
              "name": "Download File",
              "type": "task",
              "disabled": !data["files"][i]["is_file"],
              "ui_feature": "file_browser:download",
              "parameters": ls_path
            }
        ]
    }
```

Notice how we have the exact same information for the `task` and `dictionary` buttons as before, but they're just in an array format now. It's as easy as that. You can even keep your logic for disabling entries or conditionally not even add them. This allows us to create a dropdown menu like the following screenshot:

![](<../../.gitbook/assets/Screen Shot 2021-11-04 at 3.03.58 PM.png>)