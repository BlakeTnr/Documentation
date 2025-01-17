# File Hosting

## What is it?

There are times you want to host a file or payload through your C2 channel at a custom endpoint. However, every C2 profile works a bit differently; so, there needs to be a way to universally tell the C2 profile to host a file in its own way at a certain endpoint.

## What does it look like?

Below is the function definition to include with your C2 profile to host a custom file.&#x20;

{% tabs %}
{% tab title="Python" %}
```python
async def host_file(self, inputMsg: C2HostFileMessage) -> C2HostFileMessageResponse:
    """Host a file through a c2 channel

    :param inputMsg: The file UUID to host and which URL to host it at
    :return: C2HostFileMessageResponse detailing success or failure to host the file
    """
    response = C2HostFileMessageResponse(Success=True)
    response.Message = "Not Implemented"
    response.Message += f"\nInput: {json.dumps(inputMsg.to_json(), indent=4)}"
    return response
```


{% endtab %}

{% tab title="Golang" %}
```go
HostFileFunction           func(message C2HostFileMessage) C2HostFileMessageResponse
```

```go
package c2structs

type C2_HOST_FILE_STATUS = string

type C2HostFileMessage struct {
   Name     string `json:"c2_profile_name"`
   FileUUID string `json:"file_uuid"`
   HostURL  string `json:"host_url"`
}

type C2HostFileMessageResponse struct {
   Success bool   `json:"success"`
   Error   string `json:"error"`
}
```
{% endtab %}
{% endtabs %}

## Where is it?

In the Mythic UI, you'll see blue globe icons that open prompts to host those files via any C2.
