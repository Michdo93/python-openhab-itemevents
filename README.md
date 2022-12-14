# Python openHAB ItemEvents
A simple program for accessing the ItemEvents of openHAB by using the server-sent events from the openHAB REST API. Whether via the cloud or locally.

`Server-Sent Events` (`SSE`) is a server push technology enabling a client to receive automatic updates from a server via an `HTTP` connection, and describes how servers can initiate data transmission towards clients once an initial client connection has been established. They are commonly used to send message updates or continuous data streams to a browser client and designed to enhance native, cross-browser streaming through a `JavaScript API` called `EventSource`, through which a client requests a particular `URL` in order to receive an event stream. The `EventSource API` is standardized as part of `HTML5` by the `WHATWG`. The mime type for `SSE` is `text/event-stream`.

The openHAB Item Events can only received by using `SSE`!

You can get more informations about the [ItemEvents here](https://www.openhab.org/docs/developer/utils/events.html#item-events).

To test the ItemEvents, you can play around a bit using [CRUD](https://github.com/Michdo93/openhab_python_crud).

## Installation

You can install it by using `pip`:

```
python3 -m pip install python-openhab-itemevents
```

## Usage

At first you have to import the module:

```
from openhab import ItemEvent
```

The `<base_url>` could be the ip address plus port of your local openHAB instance (`<ip_address>:<port>`) or `myopenhab.org`. Of course if you are using `http` on your local instance you have to replace `https` with `http`.

|  Event | 	Description | 	URL | 
|:-------------:| :-----:| :-----:|
| ItemAddedEvent |	An item has been added to the item registry. |	curl "https://<base_url>/rest/events?topics=openhab/items/{itemName}/added" |
| ItemRemovedEvent |	An item has been removed from the item registry. | curl	"https://<base_url>/rest/events?topics=openhab/items/{itemName}/removed" |
| ItemUpdatedEvent |	An item has been updated in the item registry. |	curl "https://<base_url>/rest/events?topics=openhab/items/{itemName}/updated" |
| ItemCommandEvent |	A command is sent to an item via a channel. |	curl "https://<base_url>/rest/events?topics=openhab/items/{itemName}/command" |
| ItemStateEvent |	The state of an item is updated. |	curl "https://<base_url>/rest/events?topics=openhab/items/{itemName}/state" |
| ItemStatePredictedEvent |	The state of an item predicted to be updated. |	curl "https://<base_url>/rest/events?topics=openhab/items/{itemName}/statepredicted" |
| ItemStateChangedEvent |	The state of an item has changed. |	curl "https://<base_url>/rest/events?topics=openhab/items/{itemName}/statechanged" |
| GroupItemStateChangedEvent |	The state of a group item has changed through a member. |	curl "https://<base_url>/rest/events?topics=openhab/items/{itemName}/{memberName}/statechanged" |

### Create connection to the openHAB Cloud (as example myopenhab.org)

If you want to create a connection to the `openHAB Cloud` you have to run:

```
response = ItemEvent("https://myopenhab.org", "<your_email>@<your_provider>", "<email_password>")
```

Please make sure to replace `<your_email>@<your_provider>` with your email address and `<email_password>` with your password that you used for your openHAB Cloud account.

### Create connection to your local openHAB instance

If you want to create a connection to your local `openHAB` instance you have to replace `<username>` and `<password>` with the username and password of your local openHAB account:

```
response = ItemEvent("http://<your_ip>:8080", "<username>", "<password>")
```

Maybe there is no username and password needed:

```
response = ItemEvent("http://<your_ip>:8080")
```

### Retrieving all Item Events

You can retrieve all `Events` from all `Items` by using:

```
response =  item_event.ItemEvent()
```

You will get a `list` of `dictionaries` (`dict`) in `JSON` format. So you have to loop through it:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

Of course you can checkt if it is the type `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(type(data))
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

It is not possible not to use a loop for events, because on each change the server pushes. The created loop receives each new event as soon as it is pushed. If no new event is triggered on the server, there is no new iteration.

For all ItemEvents, therefore, there must be both a continuous loop and the data must be converted from `JSON` to a `dict`!

Of course, you can also access individual elements of the `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                topic = data.get("topic")
                payload = json.loads(data.get("payload"))
                event_type = payload.get("type")
                event_value = payload.get("value")
                item_event_type = data.get("type")

                print(topic)
                print(payload)
                print(event_type)
                print(event_value)
                print(item_event_type)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

However, `ItemEvents` do not provide information about which type an `Item` has. An `ItemEvent` contains only the changes for an `Item` that are described by an `Event`. But you can also use the `ItemEvent` to query an `Item`. This does not require `SSE`, but a simple `REST` request:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                topic = data.get("topic")
                event_item_name = topic.split("/")[2]

                try:
                    item_response = response.session.get("http://<your_ip>:8080/rest/items/" + event_item_name, auth=(<auth>), headers={"Content-type": "application/json"}, timeout=8)
                    item_response.raise_for_status()

                    if item_response.ok or item_response.status_code == 200:
                        item = json.loads(item_response.text)
                        item_link = item.get("link")
                        item_state = item.get("state")
                        item_state_description = item.get("stateDescription")
                        item_editable = item.get("editable")
                        item_type = item.get("type")
                        item_name = item.get("name")
                        item_label = item.get("label")
                        item_group_names = item.get("groupNames")

                        print(item_type)
                        print(item_name)
                        print(item_state)
                except requests.exceptions.HTTPError as errh:
                    print(errh)
                except requests.exceptions.ConnectionError as errc:
                    print(errc)
                except requests.exceptions.Timeout as errt:
                    print(errt)
                except requests.exceptions.RequestException as err:
                    print(err)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

What is missing in this pseudo code now is, `<base_url>`. You could use this for the cloud or for the local instance. A better approach is to use [CRUD](https://github.com/Michdo93/openhab_python_crud):

```
crud = CRUD("http://<your_ip>:8080", "<username>", "<password>")
item_event = ItemEvent("http://<your_ip>:8080")
response = item_event.ItemStateEvent()

with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                topic = data.get("topic")
                event_item_name = topic.split("/")[2]

                state = crud.getState(event_item_name)
                print(state)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

In the above code you will see that I used the `ItemStateEvent` because it is a little bit faster than `ItemEvent`. And you can mirror ot to your `mqtt broker` if you want to:

```
from openhab_ItemEvents import ItemEvent
from openHAB_CRUD import CRUD
import json
import paho.mqtt.client as mqtt
import string
import random

# Main function.
if __name__ == "__main__":
    client_pre = "openHAB"
    client_post = "".join(random.choice(string.ascii_uppercase + string.digits) for _ in range(3))
    client_name = client_pre[:20] + client_post

    client = mqtt.Client(client_id=client_name, clean_session=True, userdata=None, protocol=mqtt.MQTTv311, transport="tcp")
    client.username_pw_set(None)
    qos = 0

    client.connect("<your_ip>", 1883)

    crud = CRUD("http://<your_ip>:8080", "<username>", "<password>")
    item_event = ItemEvent("http://<your_ip>:8080:8080")
    response = item_event.ItemStateEvent()

    with response as events:
        for line in events.iter_lines():
            line = line.decode()

            if "data" in line:
                line = line.replace("data: ", "")

                try:
                    data = json.loads(line)
                    topic = data.get("topic")
                    event_item_name = topic.split("/")[2]

                    state = crud.getState(event_item_name)
                    client.publish(f"openhab/item/{event_item_name}/state", payload=state, qos=qos, retain=False)
                    print(f"openhab/item/{event_item_name}/state {state}" )
                except json.decoder.JSONDecodeError:
                    print("Event could not be converted to JSON")
```

Since you get all ItemEvents` for all `Items`, it is recommended to make a case distinction:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                topic = data.get("topic")
                payload = json.loads(data.get("payload"))
                event_type = payload.get("type")

                if(event_type == "ItemAddedEvent"):
                    # your code
                elif(event_type == "ItemRemovedEvent"):
                    # your code
                elif(event_type == "ItemUpdatedEvent"):
                    # your code
                elif(event_type == "ItemCommandEvent"):
                    # your code
                elif(event_type == "ItemStateEvent"):
                    # your code
                elif(event_type == "ItemStatePredictedEvent"):
                    # your code
                elif(event_type == "ItemStateChangedEvent"):
                    # your code
                elif(event_type == "GroupItemStateChangedEvent"):
                    # your code
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")  
```

Of course, it is possible to query not only all `ItemEvents`, but also individual `ItemEvents` for an `Item` or individual `ItemEvents` for all `Items`.

### Retrieving ItemAddedEvent of an Item

The function requires the name of the item as input parameter:

```
response =  item_event.ItemAddedEvent("testItem")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemAddedEvent of all Items

The function does not require a parameter:

```
response =  item_event.ItemAddedEvent()
```

Alternatively you can pass a `"*"`:

```
response =  item_event.ItemAddedEvent("*")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemRemovedEvent of an Item

The function requires the name of the item as input parameter:

```
response =  item_event.ItemRemovedEvent("testItem")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemRemovedEvent of all Items

The function does not require a parameter:

```
response =  item_event.ItemRemovedEvent()
```

Alternatively you can pass a `"*"`:

```
response =  item_event.ItemRemovedEvent("*")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemUpdatedEvent of an Item

The function requires the name of the item as input parameter:

```
response =  item_event.ItemUpdatedEvent("testItem")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemUpdatedEvent of all Items

The function does not require a parameter:

```
response =  item_event.ItemUpdatedEvent()
```

Alternatively you can pass a `"*"`:

```
response =  item_event.ItemUpdatedEvent("*")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemCommandEvent of an Item

The function requires the name of the item as input parameter:

```
response =  item_event.ItemCommandEvent("testItem")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemCommandEvent of all Items

The function does not require a parameter:

```
response =  item_event.ItemCommandEvent()
```

Alternatively you can pass a `"*"`:

```
response =  item_event.ItemCommandEvent("*")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemStateEvent of an Item

The function requires the name of the item as input parameter:

```
response =  item_event.ItemStateEvent("testItem")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemStateEvent of all Items

The function does not require a parameter:

```
response =  item_event.ItemStateEvent()
```

Alternatively you can pass a `"*"`:

```
response =  item_event.ItemStateEvent("*")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemStatePredictedEvent of an Item

The function requires the name of the item as input parameter:

```
response =  item_event.ItemStatePredictedEvent("testItem")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemStatePredictedEvent of all Items

The function does not require a parameter:

```
response =  item_event.ItemStatePredictedEvent()
```

Alternatively you can pass a `"*"`:

```
response =  item_event.ItemStatePredictedEvent("*")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemStateChangedEvent of an Item

The function requires the name of the item as input parameter:

```
response =  item_event.ItemStateChangedEvent("testItem")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving ItemStateChangedEvent of all Items

The function does not require a parameter:

```
response =  item_event.ItemStateChangedEvent()
```

Alternatively you can pass a `"*"`:

```
response =  item_event.ItemStateChangedEvent("*")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```

### Retrieving GroupItemStateChangedEvent of an Item

The function requires the names of the item and of the member as input parameter:

```
response =  item_event.GroupItemStateChangedEvent("testItem", "testGroup")
```

Finally, in the end you have to remember that you need to convert from JSON and you need a loop. You get again a `dict`:

```
with response as events:
    for line in events.iter_lines():
        line = line.decode()

        if "data" in line:
            line = line.replace("data: ", "")

            try:
                data = json.loads(line)
                print(data)
            except json.decoder.JSONDecodeError:
                print("Event could not be converted to JSON")
```
