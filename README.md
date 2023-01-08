Intro: 
TODO:


## Build from scratch:

---
### Initialize project
* `django-admin startproject mysite`
* `python manage.py startapp chat`
* add `chat` to installed apps
* create `templates/chat/lobby.html` in `/chat` folder
* set up urls/views for `lobby` (single page site)
* `python manage.py runserver` to test

---
### Adding base websocket connection
* `channels==3.0.4`
* add `channels` to installed apps list
* add script to `lobby.html` to log messages from server:
```html
    <script type="text/javascript">
        let url = `ws://${window.location.host}/ws/socket-server/`
        const chatSocket = new WebSocket(url)
        chatSocket.onopen = function (e){
            console.log("I see connection on my side. Waiting for messages...")
        }
         chatSocket.onmessage = function (e){
            console.log("I got message!!")
            let data = JSON.parse(e.data)
            console.log(data)
        }
    </script>
```
* create `chat/consumers.py`. It would be a file like views, but for more advanced operations
* add class for handling events:
```python
import json
from channels.generic.websocket import WebsocketConsumer

class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.accept()
        self.send(text_data=json.dumps({
            "type": "connection_established",
            "message": "You are connected!"
        }))
```
* connect it in `chat/routing.py`:
```python
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r"ws/socket-server/", consumers.ChatConsumer.as_asgi())
]
```
* replace code in `asgi.py` with next:
```python
import os

from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
import chat.routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mysite.settings')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(
            chat.routing.websocket_urlpatterns
        )
    )
})

```
* add `ASGI_APPLICATION = 'mysite.asgi.application'` to `settings.py`

* Run migrations `python manage.py migrate`
* Rerun server and check console on the UI. connection_established message should be there

---

### Adding 2 sides communication to chat

* Add form to `lobby.html`:
```html
    <form id="form">
        <input type="text" name="message">
    </form>
```

* Handle message sending from form in `<script>`::
```javascript
    let form = document.getElementById("form")
    form.addEventListener("submit", (e)=> {
        e.preventDefault()
        let message = e.target.message.value
        chatSocket.send(JSON.stringify({
            "message": message
        }))
        form.reset()
    })
```

* Add logic of message receiving to `ChatConsumer`:
```python
    def receive(self, text_data=None, bytes_data=None):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"]

        print(message)
        self.send(text_data=json.dumps({
            "type": "chat",
            "message": message
        }))
```

* Add element for holding chat in `lobby.html`:
```html
    <div id="messages"></div>
```
* And handle filling this element in script section:
```js
         chatSocket.onmessage = function (e){
            console.log("I got message!!")
            let data = JSON.parse(e.data)
            console.log(data)
            
            // New code
            if (data.type === "chat"){
                let messages = document.getElementById("messages")
                messages.insertAdjacentHTML("beforeend", `
                    <div><p>${data.message}</p></div>
                `)
            }
            // end New code
        }
```
Now we have 2 sides communication with **single** client and server.

---

