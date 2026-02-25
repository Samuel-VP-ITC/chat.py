# chat.py
Chat básico de red local con perfiles diferentes y hecho con herramientas de Flet
Añadir controles de página y gestionar eventos#
Para empezar, queremos poder tomar la entrada del usuario (mensaje de chat) y mostrar el historial de mensajes en pantalla. La disposición de este paso podría ser la siguiente:

chat-layout-1

Para implementar este diseño, utilizaremos estos controles Flet:

Columna: un contenedor para mostrar los mensajes de chat (controles) en vertical.Text
Mensaje de texto en el chat mostrado en la columna de chat.
Fila: un contenedor para exponer y horizontalmente.TextFieldButton
TextField - control de entrada utilizado para recibir nuevas entradas de mensajes del usuario.
Botón - Botón "Enviar" que añadirá un nuevo mensaje a la columna de chat.
Crea con los siguientes contenidos:chat.py


import flet as ft


def main(page: ft.Page):
    chat = ft.Column()
    new_message = ft.TextField()

    def send_click(e):
        chat.controls.append(ft.Text(new_message.value))
        new_message.value = ""

    page.add(
        chat,
        ft.Row(controls=[new_message, ft.Button("Send", on_click=send_click)]),
    )


ft.run(main)
Cuando el usuario hace clic en el botón "Enviar", se activa on_click evento que llama a método. luego añade un nuevo control de texto a la lista de Column.controls y borra el valor del campo de texto.send_clicksend_clicknew_message

Nota

Después de que se actualicen las propiedades de un control en el gestor de eventos, no es necesario llamar a un método, ya que la actualización se hará automáticamente, pero debe llamarse en caso de que las propiedades se actualicen fuera del evento.update()

La app de chat ahora se ve así:chat-1

Transmisión de mensajes de chat#
En el paso anterior hemos creado una aplicación sencilla que acepta la entrada del usuario y muestra los mensajes de chat en pantalla.

Si abres esta app en dos pestañas del navegador web, creará dos sesiones de aplicación. Cada sesión tendrá su propia lista de mensajes.

Propina

Para abrir tu aplicación en dos pestañas del navegador web localmente, ejecuta el siguiente comando:


flet run --web <path_to_your_app>
Una vez abierto, copia la URL y pégala en una pestaña nueva.
Para construir una app de chat en tiempo real, necesitas de alguna manera pasar los mensajes entre las sesiones de la app de chat. Cuando un usuario envía un mensaje, este debe transmitirse a todas las demás sesiones de la app y mostrarse en sus páginas.

Flet proporciona un mecanismo PubSub integrado sencillo para la comunicación asíncrona entre sesiones de página.

Primero, necesitamos suscribirse al usuario para recibir mensajes de difusión:


    page.pubsub.subscribe(on_message)
pubsub.subscribe() Method añadirá la sesión actual de la app a la lista de suscriptores. Acepta como argumento que luego se llamará en el momento en que un editor llame a la orden.handlerpubsub.send_all()

En el vamos a añadir un nuevo mensaje () a la lista de chat: handlerTextcontrols


    def on_message(message: Message):
        chat.controls.append(ft.Text(f"{message.user}: {message.text}"))
        page.update()
Finalmente, necesitas llamar al método cuando el usuario haga clic en el botón "Enviar": pubsub.send_all()


    def send_click(e):
        page.pubsub.send_all(Message(user=page.session.index, text=new_message.value))
        new_message.value = ""
        page.update()

    page.add(chat, ft.Row([new_message, ft.Button("Send", on_click=send_click)]))
pubsub.send_all() llamará a y pasará el objeto Mensaje hacia él.on_message()

Aquí tienes el código completo de este paso:


from dataclasses import dataclass

import flet as ft


@dataclass
class Message:
    user: str
    text: str


def main(page: ft.Page):
    chat = ft.Column()
    new_message = ft.TextField()

    def on_message(message: Message):
        chat.controls.append(ft.Text(f"{message.user}: {message.text}"))
        page.update()

    page.pubsub.subscribe(on_message)

    def send_click(e):
        page.pubsub.send_all(Message(user=page.session.id, text=new_message.value))
        new_message.value = ""

    page.add(chat, ft.Row([new_message, ft.Button("Send", on_click=send_click)]))


ft.run(main)
chat-2

User name dialog#
Chat app that you have created in the previous step has basic functionality needed to exchange messages between user sessions. It is not very user-friendly though, since it shows that sent a message, which doesn't tell much about who you are communicating with.session.id

Let's improve our app to show user name instead of for each message. To capture user name we will be using AlertDialog control. Let's add it to the page:session.id


    user_name = ft.TextField(label="Enter your name")

    page.show_dialog(
        ft.AlertDialog(
            open=True,
            modal=True,
            title=ft.Text("Welcome!"),
            content=ft.Column([user_name], tight=True),
            actions=[ft.Button(content="Join chat", on_click=join_click)],
            actions_alignment=ft.MainAxisAlignment.END,
        )
    )
Nota

Se abrirá un diálogo al inicio del programa, ya que hemos establecido su propiedad en .openTrue

username-dialog

Cuando el usuario haga clic en el botón "Unirse al chat", llamará a un método que enviará un mensaje a todos los suscriptores, informándoles de que el usuario se ha unido al chat. Este mensaje debería verse diferente al chat normal, por ejemplo, así:join_click

chat-4

Vamos a añadir una propiedad a la clase para diferenciar entre mensajes de inicio de sesión y mensajes de chat:message_typeMessage


@dataclass
class Message:
    user: str
    text: str
    message_type: str
Vamos a revisar el método:message_typeon_message


def on_message(message: Message):
    if message.message_type == "chat_message":
        chat.controls.append(ft.Text(f"{message.user}: {message.text}"))
    elif message.message_type == "login_message":
        chat.controls.append(
            ft.Text(message.text, italic=True, color=ft.Colors.BLACK_45, size=12)
        )
    page.update()
Ahora se enviarán mensajes de tipo "login_message" y "chat_message" en dos eventos: cuando el usuario se une al chat y cuando el usuario envía un mensaje.

Vamos a crear un método:join_click


def join_click(e):
    if not user_name.value:
        user_name.error = "Name cannot be blank!"
        user_name.update()
    else:
        page.session.store.set("user_name", user_name.value)
        page.pop_dialog()
        page.pubsub.send_all(Message(user=user_name.value, text=f"{user_name.value} has joined the chat.", message_type="login_message"))
We used page session storage to store user_name for its future use in method to send chat messages.send_click

Note

User name dialog will close as soon as we call method.page.pop_dialog()

Finally, let's update method to use that we previously saved using :send_clickuser_namepage.session.store


def send_click(e):
    page.pubsub.send_all(Message(user=page.session.store.get('user_name'), text=new_message.value, message_type="chat_message"))
    new_message.value = ""
Code

from dataclasses import dataclass

import flet as ft


@dataclass
class Message:
    user: str
    text: str
    message_type: str


def main(page: ft.Page):
    chat = ft.Column()
    new_message = ft.TextField()

    def on_message(message: Message):
        if message.message_type == "chat_message":
            chat.controls.append(ft.Text(f"{message.user}: {message.text}"))
        elif message.message_type == "login_message":
            chat.controls.append(
                ft.Text(message.text, italic=True, color=ft.Colors.BLACK_45, size=12)
            )
        page.update()

    page.pubsub.subscribe(on_message)

    def send_click(e):
        page.pubsub.send_all(
            Message(
                user=page.session.store.get("user_name"),
                text=new_message.value,
                message_type="chat_message",
            )
        )
        new_message.value = ""

    user_name = ft.TextField(label="Enter your name")

    def join_click(e):
        if not user_name.value:
            user_name.error_text = "Name cannot be blank!"
        else:
            page.session.store.set("user_name", user_name.value)
            # page.dialog.open = False
            page.pop_dialog()
            page.pubsub.send_all(
                Message(
                    user=user_name.value,
                    text=f"{user_name.value} has joined the chat.",
                    message_type="login_message",
                )
            )

    page.show_dialog(
        ft.AlertDialog(
            open=True,
            modal=True,
            title=ft.Text("Welcome!"),
            content=ft.Column([user_name], tight=True),
            actions=[ft.Button(content="Join chat", on_click=join_click)],
            actions_alignment=ft.MainAxisAlignment.END,
        )
    )

    page.add(chat, ft.Row([new_message, ft.Button("Send", on_click=send_click)]))


ft.run(main)
chat-3

Enhancing user interface#
Chat app that you have created in the previous step already serves its purpose of exchanging messages between users with basic login functionality.

Before moving on to deploying your app, we suggest adding some extra features to it that will improve user experience and make the app look more professional.

Reusable user controls#
You may want to show messages in a different format, like this:

chat-layout-chatmessage

Chat message will now be a Row containing CircleAvatar with username initials and Column that contains two Text controls: user name and message text.

We will need to show quite a few chat messages in the chat app, so it makes sense to create your own reusable control. Lets create a new dataclass that will inherit from Row.ChatMessage

When creating an instance of class, we will pass a object as an argument and then will display itself based on and :ChatMessageMessageChatMessagemessage.user_namemessage.text


@ft.control
class ChatMessage(ft.Row):
    def __init__(self, message: Message):
        super().__init__()
        self.message = message
        self.vertical_alignment = ft.CrossAxisAlignment.START
        self.controls = [
            ft.CircleAvatar(
                content=ft.Text(self.get_initials(self.message.user_name)),
                color=ft.Colors.WHITE,
                bgcolor=self.get_avatar_color(self.message.user_name),
            ),
            ft.Column(
                tight=True,
                spacing=5,
                controls=[
                    ft.Text(self.message.user_name, weight=ft.FontWeight.BOLD),
                    ft.Text(self.message.text, selectable=True),
                ],
            ),
        ]

    def get_initials(self, user_name: str):
        if user_name:
            return user_name[:1].capitalize()
        else:
            return "Unknown"  # or any default value you prefer

    def get_avatar_color(self, user_name: str):
        colors_lookup = [
            ft.Colors.AMBER,
            ft.Colors.BLUE,
            ft.Colors.BROWN,
            ft.Colors.CYAN,
            ft.Colors.GREEN,
            ft.Colors.INDIGO,
            ft.Colors.LIME,
            ft.Colors.ORANGE,
            ft.Colors.PINK,
            ft.Colors.PURPLE,
            ft.Colors.RED,
            ft.Colors.TEAL,
            ft.Colors.YELLOW,
        ]
        return colors_lookup[hash(user_name) % len(colors_lookup)]
ChatMessage control extracts initials and algorithmically derives avatar color from a username. Later, if you decide to improve control layout or its logic, it won't affect the rest of the program - that's the power of encapsulation!
Laying out controls#
Now you can use your brand new to build a better layout for the chat app:ChatMessage

chat-layout-2

Instances of will be created instead of plain chat messages in method:ChatMessageTexton_message


    def on_message(message: Message):
        if message.message_type == "chat_message":
            m = ChatMessage(message)
        elif message.message_type == "login_message":
            m = ft.Text(message.text, italic=True, color=ft.Colors.BLACK_45, size=12)
        chat.controls.append(m)
        page.update()
Other improvements suggested with the new layout are:

ListView instead of Column for displaying messages, to be able to scroll through the messages later
Container for displaying border around ListView
IconButton instead of Button to send messages
Use of expand property for controls to fill available space
Here is how you can implement this layout:


    # Chat messages
    chat = ft.ListView(
        expand=True,
        spacing=10,
        auto_scroll=True,
    )

    # A new message entry form
    new_message = ft.TextField(
        hint_text="Write a message...",
        autofocus=True,
        shift_enter=True,
        min_lines=1,
        max_lines=5,
        filled=True,
        expand=True,
        on_submit=send_message_click,
    )

    # Add everything to the page
    page.add(
        ft.Container(
            content=chat,
            border=ft.border.all(1, ft.Colors.OUTLINE),
            border_radius=5,
            padding=10,
            expand=True,
        ),
        ft.Row(
            [
                new_message,
                ft.IconButton(
                    icon=ft.Icons.SEND_ROUNDED,
                    tooltip="Send message",
                    on_click=send_message_click,
                ),
            ]
        ),
    )
Full code

from dataclasses import dataclass

import flet as ft


@dataclass
class Message:  # noqa: B903
    user_name: str
    text: str
    message_type: str


@ft.control
class ChatMessage(ft.Row):
    def __init__(self, message: Message):
        super().__init__()
        self.message = message
        self.vertical_alignment = ft.CrossAxisAlignment.START
        self.controls = [
            ft.CircleAvatar(
                content=ft.Text(self.get_initials(self.message.user_name)),
                color=ft.Colors.WHITE,
                bgcolor=self.get_avatar_color(self.message.user_name),
            ),
            ft.Column(
                tight=True,
                spacing=5,
                controls=[
                    ft.Text(self.message.user_name, weight=ft.FontWeight.BOLD),
                    ft.Text(self.message.text, selectable=True),
                ],
            ),
        ]

    def get_initials(self, user_name: str):
        if user_name:
            return user_name[:1].capitalize()
        else:
            return "Unknown"  # or any default value you prefer

    def get_avatar_color(self, user_name: str):
        colors_lookup = [
            ft.Colors.AMBER,
            ft.Colors.BLUE,
            ft.Colors.BROWN,
            ft.Colors.CYAN,
            ft.Colors.GREEN,
            ft.Colors.INDIGO,
            ft.Colors.LIME,
            ft.Colors.ORANGE,
            ft.Colors.PINK,
            ft.Colors.PURPLE,
            ft.Colors.RED,
            ft.Colors.TEAL,
            ft.Colors.YELLOW,
        ]
        return colors_lookup[hash(user_name) % len(colors_lookup)]


def main(page: ft.Page):
    page.horizontal_alignment = ft.CrossAxisAlignment.STRETCH
    page.title = "Flet Chat"

    def join_chat_click(e):
        if not join_user_name.value:
            join_user_name.error_text = "Name cannot be blank!"
            join_user_name.update()
        else:
            page.session.store.set("user_name", join_user_name.value)
            welcome_dlg.open = False
            new_message.prefix = ft.Text(f"{join_user_name.value}: ")
            page.pubsub.send_all(
                Message(
                    user_name=join_user_name.value,
                    text=f"{join_user_name.value} has joined the chat.",
                    message_type="login_message",
                )
            )

    async def send_message_click(e):
        if new_message.value != "":
            page.pubsub.send_all(
                Message(
                    page.session.store.get("user_name"),
                    new_message.value,
                    message_type="chat_message",
                )
            )
            new_message.value = ""
            await new_message.focus()

    def on_message(message: Message):
        if message.message_type == "chat_message":
            m = ChatMessage(message)
        elif message.message_type == "login_message":
            m = ft.Text(message.text, italic=True, color=ft.Colors.BLACK_45, size=12)
        chat.controls.append(m)
        page.update()

    page.pubsub.subscribe(on_message)

    # A dialog asking for a user display name
    join_user_name = ft.TextField(
        label="Enter your name to join the chat",
        autofocus=True,
        on_submit=join_chat_click,
    )
    welcome_dlg = ft.AlertDialog(
        open=True,
        modal=True,
        title=ft.Text("Welcome!"),
        content=ft.Column([join_user_name], width=300, height=70, tight=True),
        actions=[ft.Button(content="Join chat", on_click=join_chat_click)],
        actions_alignment=ft.MainAxisAlignment.END,
    )

    page.overlay.append(welcome_dlg)

    # Chat messages
    chat = ft.ListView(
        expand=True,
        spacing=10,
        auto_scroll=True,
    )

    # A new message entry form
    new_message = ft.TextField(
        hint_text="Write a message...",
        autofocus=True,
        shift_enter=True,
        min_lines=1,
        max_lines=5,
        filled=True,
        expand=True,
        on_submit=send_message_click,
    )

    # Add everything to the page
    page.add(
        ft.Container(
            content=chat,
            border=ft.Border.all(1, ft.Colors.OUTLINE),
            border_radius=5,
            padding=10,
            expand=True,
        ),
        ft.Row(
            controls=[
                new_message,
                ft.IconButton(
                    icon=ft.Icons.SEND_ROUNDED,
                    tooltip="Send message",
                    on_click=send_message_click,
                ),
            ]
        ),
    )


ft.run(main)
Esta es la versión final de la app de chat para este tutorial.

´´´
from dataclasses import dataclass

import flet as ft


@dataclass
class Message:  # noqa: B903
    user_name: str
    text: str
    message_type: str


@ft.control
class ChatMessage(ft.Row):
    def __init__(self, message: Message):
        super().__init__()
        self.message = message
        self.vertical_alignment = ft.CrossAxisAlignment.START
        self.controls = [
            ft.CircleAvatar(
                content=ft.Text(self.get_initials(self.message.user_name)),
                color=ft.Colors.WHITE,
                bgcolor=self.get_avatar_color(self.message.user_name),
            ),
            ft.Column(
                tight=True,
                spacing=5,
                controls=[
                    ft.Text(self.message.user_name, weight=ft.FontWeight.BOLD),
                    ft.Text(self.message.text, selectable=True),
                ],
            ),
        ]

    def get_initials(self, user_name: str):
        if user_name:
            return user_name[:1].capitalize()
        else:
            return "Unknown"  # or any default value you prefer

    def get_avatar_color(self, user_name: str):
        colors_lookup = [
            ft.Colors.AMBER,
            ft.Colors.BLUE,
            ft.Colors.BROWN,
            ft.Colors.CYAN,
            ft.Colors.GREEN,
            ft.Colors.INDIGO,
            ft.Colors.LIME,
            ft.Colors.ORANGE,
            ft.Colors.PINK,
            ft.Colors.PURPLE,
            ft.Colors.RED,
            ft.Colors.TEAL,
            ft.Colors.YELLOW,
        ]
        return colors_lookup[hash(user_name) % len(colors_lookup)]


def main(page: ft.Page):
    page.horizontal_alignment = ft.CrossAxisAlignment.STRETCH
    page.title = "Flet Chat"

    def join_chat_click(e):
        if not join_user_name.value:
            join_user_name.error_text = "Name cannot be blank!"
            join_user_name.update()
        else:
            page.session.store.set("user_name", join_user_name.value)
            welcome_dlg.open = False
            new_message.prefix = ft.Text(f"{join_user_name.value}: ")
            page.pubsub.send_all(
                Message(
                    user_name=join_user_name.value,
                    text=f"{join_user_name.value} has joined the chat.",
                    message_type="login_message",
                )
            )

    async def send_message_click(e):
        if new_message.value != "":
            page.pubsub.send_all(
                Message(
                    page.session.store.get("user_name"),
                    new_message.value,
                    message_type="chat_message",
                )
            )
            new_message.value = ""
            await new_message.focus()

    def on_message(message: Message):
        if message.message_type == "chat_message":
            m = ChatMessage(message)
        elif message.message_type == "login_message":
            m = ft.Text(message.text, italic=True, color=ft.Colors.BLACK_45, size=12)
        chat.controls.append(m)
        page.update()

    page.pubsub.subscribe(on_message)

    # A dialog asking for a user display name
    join_user_name = ft.TextField(
        label="Enter your name to join the chat",
        autofocus=True,
        on_submit=join_chat_click,
    )
    welcome_dlg = ft.AlertDialog(
        open=True,
        modal=True,
        title=ft.Text("Welcome!"),
        content=ft.Column([join_user_name], width=300, height=70, tight=True),
        actions=[ft.Button(content="Join chat", on_click=join_chat_click)],
        actions_alignment=ft.MainAxisAlignment.END,
    )

    page.overlay.append(welcome_dlg)

    # Chat messages
    chat = ft.ListView(
        expand=True,
        spacing=10,
        auto_scroll=True,
    )

    # A new message entry form
    new_message = ft.TextField(
        hint_text="Write a message...",
        autofocus=True,
        shift_enter=True,
        min_lines=1,
        max_lines=5,
        filled=True,
        expand=True,
        on_submit=send_message_click,
    )

    # Add everything to the page
    page.add(
        ft.Container(
            content=chat,
            border=ft.Border.all(1, ft.Colors.OUTLINE),
            border_radius=5,
            padding=10,
            expand=True,
        ),
        ft.Row(
            controls=[
                new_message,
                ft.IconButton(
                    icon=ft.Icons.SEND_ROUNDED,
                    tooltip="Send message",
                    on_click=send_message_click,
                ),
            ]
        ),
    )


#ft.run(main)
ft.app(target=main, view=ft.AppView.WEB_BROWSER)
´´´
