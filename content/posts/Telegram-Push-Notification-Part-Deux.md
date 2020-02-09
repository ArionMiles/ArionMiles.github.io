---
title: Building a Push Notification service for Telegram Bots (PART 2)
date: 2020-02-08
type: "post"
description: "Continued from previous blog post about push notifications for Telegram bots"
tags: 
 - telegram 
 - python
---

In the previous post, we came up with a multi-threaded asynchronous way to send push notifications to more than 
300 users as quickly as possible. Since the last article, I've gone from 300+ users to ~700 users, and the solution 
is still standing.

In this blogpost, we'll discuss how to record all push notifications you send to our users into a database and how 
it can help you in learning how many users we're actually reaching, and reason out why some messages didn't reach 
our users. Additionally, we'll be building a nifty "undo" feature so we can erase accidental messages.

In this blogpost, we'll not be using the HTTP Telegram Bot API, but instead, a neat little wrapper library called 
`python-telegram-bot` (henceforth referred to as `ptb`) which brings brevity to the code.


## Using python-telegram-bot

We'll be using the `python-telegram-bot` module, which is a wrapper over the Telegram Bot HTTP API. To install it, run:

```
pip install python-telegram-bot
```

This is our push notification feature we came up with in our previous blog:

```python
import os
from functools import partial
import concurrent.futures
import requests

API_KEY_TOKEN = os.getenv('API_KEY_TOKEN', 'default_token_value')

def push_threaded(message, user_list):
    push_func = partial(push_t, message)
    with concurrent.futures.ThreadPoolExecutor(max_workers=30) as executor:
        executor.map(push_func, user_list)

def push_t(message, chat_id):
    url = "https://api.telegram.org/bot{}/sendMessage".format(API_KEY_TOKEN)

    payload = {"text": message,
               "chat_id": chat_id, 
               "parse_mode": "markdown"
    }
    
    response = requests.post(url, data=payload)
    return response.text
```

We'll make a change in our `push_t()` function, to take advantage of `ptb`'s `telegram.bot.Bot` class.

```python
import os
import concurrent.futures
import threading
from functools import partial

from telegram.bot import Bot
from telegram.utils.request import Request

API_KEY_TOKEN = os.getenv('API_KEY_TOKEN', 'default_token_value')


def get_bot():
    request = Request(con_pool_size=30) # Increasing default connection pool from 1
    bot = Bot(API_KEY_TOKEN, request=request)
    return bot


def push_message_threaded(message, user_list):
    push = partial(push_t, get_bot(), message)

    with concurrent.futures.ThreadPoolExecutor(max_workers=30) as executor:
        executor.map(push, user_list)


def push_t(bot, message, chat_id):
    response = bot.sendMessage(chat_id=chat_id, text=message, parse_mode='markdown')
    return response
```

You'll notice the `get_bot()` function is something different. This is because if we instantiate a Bot object with defaults like this:

```python
bot = Bot(API_KEY_TOKEN)
```
the ptb module uses a connection pool of 1 by default. This will prevent us from making more than one concurrent request in our threaded approach. To solve this, I went digging around the source code of `telegram.bot.Bot` class and found that it uses a `Request` object which defaults to a connection pool size of 1, which I then changed by passing my own `Request` object with a pool size of 30, since we don't want more than 30 concurrent requests at any instance, [as explained in Part 1](https://arionmiles.me/posts/telegram-push-notification-service/). This enables us to use ptb's fancy `Bot` object without running into concurrency problems.

Now, we can send our text messages through the simple `sendMessage` method. The Bot object allows us to send documents or any kind of media file, by utilizing [its wide range of methods](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Code-snippets#working-with-files-and-media). This will also make it simple for us to delete messages later.


## Recording Notifications into Database
> I'll only be mentioning the DB Schema I use for notifications sent out. The details of the implementation are left up to the user, though I personally use SQLAlchemy, and the model code shown below are for SQLAlchemy.

Below is the database model for storing the message that is sent out. 
- Each message is identified by a UUID (Universally Unique ID)
- The actual text that's sent out (text limit is kept at 4096 since that's the max char limit supported by Telegram)
- A boolean indicating whether this message has been deleted (will be useful later)
- A datetime field specifying the time at which the message was created.


```python
import uuid


def generate_uuid():
    return str(uuid.uuid4())


class PushMessage(Base):
    __tablename__ = 'pushmessages'
    uuid = Column(String(512), primary_key=True, default=generate_uuid)
    text = Column(String(4096))
    deleted = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.now)
    
    def __init__(self, text, created_at=None):
        self.text = text
        self.created_at = created_at

    def __repr__(self):
        return '<{} | {}>'.format(self.uuid, self.text)
```

Below is the model for storing info about whether each user received the message or not. This will be useful when we 
need to delete messages sent out accidentally.

- UUID for the message (from the previous model, could be a foreign key to `PushMessage` model as well)
- Chat ID of the user who was sent the message
- Message ID returned for the notification message sent to the user, this is useful when we wish to delete the message.
- A boolean value indicating whether the message was sent successfully. If the user has blocked the bot or deleted their telegram account, this value will be set to false.
- The text from the exception that's raised when the message fails to get delivered.

```python
class PushNotification(Base):
    __tablename__ = 'pushnotifications'
    id = Column(Integer, primary_key=True)
    message_uuid = Column(String(512))
    chat_id = Column(String(512))
    message_id = Column(Integer, default=0)
    sent = Column(Boolean, default=False)
    failure_reason = Column(String(512))

    def __init__(self, message_uuid, chat_id, message_id, sent, failure_reason=None):
        self.message_uuid = message_uuid
        self.chat_id = chat_id
        self.message_id = message_id
        self.sent = sent
        self.failure_reason = failure_reason

    def __repr__(self):
        return '<Push Notification {} | {}>'.format(self.chat_id, self.message_id)
```

### Updating our function to create notification records

We'll update the function we've previously written so that we create a database record of the push message as well as a record of message_ids sent out to each user.

```python
import os
import concurrent.futures
import threading
from functools import partial

from telegram.bot import Bot
from telegram.utils.request import Request

API_KEY_TOKEN = os.getenv('API_KEY_TOKEN', 'default_token_value')
message_records = []
inactive_users = []


def push_message_threaded(message, user_list):
    push_message = PushMessage(text=message)
    db_session.add(push_message)
    db_session.commit()
    
    message_uuid = push_message.uuid
    push = partial(push_t, get_bot(), message, message_uuid) # Adding message string and uuid as function parameter
    
    start = time.time()
    # ...
    # Using ThreadPoolExecutor to send push messages
    # ...
    elapsed = time.time() - start
    
    db_session.bulk_save_objects(message_records)
    db_session.commit()

    delete_users = Chat.__table__.delete().where(Chat.chatID.in_(inactive_users))
    db_session.execute(delete_users)
    db_session.commit()


    message_records.clear()
    inactive_users.clear()
    
    return elapsed, message_uuid
 
def push_t(bot, message, message_uuid, chat_id):
    try:
        response = bot.sendMessage(chat_id=chat_id, text=message, parse_mode='markdown')
        push_message_record = PushNotification(message_uuid=message_uuid, chatID=chat_id, message_id=response.message_id, sent=True)
        message_records.append(push_message_record)
    except Exception as e:
        push_message_record = PushNotification(message_uuid=message_uuid, chatID=chat_id, failure_reason=str(e))
        message_records.append(push_message_record)
        inactive_users.append(chat_id)
```

We make use of two global lists here `list_of_records` & `inactive_users`. These are used inside the `push_t()` function. The former list contains `PushNotification` records with `message_uuid`, `chat_id`, the `message_id`. The latter contains chat_ids of users for whom attempting to send the message resulted in an exception. We record the reason for failure (usually the user either blocked the bot or deleted their Telegram account) along with the `message_id` and the user's `chat_id`.

The `list_of_records` list contains `PushNotification` instances, which are saved in bulk to the database at the end when the notification has been sent to all users . Bulk saving at the end helps us reduce the number of queries we have to make with the database and reduce the time taken to send out the notification since our priority is sending them as fast as possible.

```python
db_session.bulk_save_objects(message_records)
db_session.commit()
```

> `db_session` is my database session object. I've not included the import statement which retrieves this object from a different file (`database.py`) which might be confusing but my focus here is on architecture and not on details of implementation, seeing as how different folks might be using different techniques to interface with their database.

The `inactive_users` list is used to delete records of inactive users. This is also done when all notifications have been sent out. You can see the lines:

```python
delete_users = Chat.__table__.delete().where(Chat.chatID.in_(inactive_users))
```

> `Chat` is the table containing my user's information (I know, bad name, but this table was created when I knew very little about programming and best practices, hopefully I'll change it one of these days) and I run a SQLAlchemy ORM query to delete all records by matching their `chatID` with the chat ids in the `inactive_users` list.

> Deleting user records is a personal choice. I prefer to do that here even though they say never hard delete records from your database. But I'm using this in a side project where it doesn't matter to me.

At the end of all this, we clear our two global lists with the below lines, so that they're empty the next time the functions are called.
```python
message_records.clear()
inactive_users.clear()
```

## The Undo Feature

Let's be honest, there will be times where we might accidently send out test messages to our users. Like this instance when PayTM, one of the popular e-wallet services in my country with more than 200 million customers sent out a push notification to everyone...

{{< figure src="https://pbs.twimg.com/media/DwK2V37XgAYgt4F?format=jpg&name=large" title="PayTM accidentally sent a test notification to all its users" >}}

They later apologized with this tweet:
{{< tweet 1081646184119250944 >}}

If we ever do this, we need to be able to save face. So I've worked up a little solution for deleting the sent messages. Remember we stored `message_id`s in our `PushNotification` model? These are necessary when we wish to delete a sent message. The Telegram Bot API requires the `chat_id` of the user to whom the message was sent, and the `message_id` identifying the particular message which we wish to delete.

We'll be deleting the messages in the same asynchronous fashion, to speed things up.

```python
def delete_threaded(message_id_list, user_list):
    delete_func = partial(delete, get_bot())
    start = time.time()
    with concurrent.futures.ThreadPoolExecutor(max_workers=30) as executor:
        executor.map(delete_func, message_id_list, user_list)
    elapsed = time.time() - start
    return elapsed


def delete(bot, message_id, chat_id):
    bot.delete_message(chat_id, message_id)
```

Each `PushNotification` record contains the `message_uuid` generated when we sent the message (which we intend to delete), the user's `chat_id` and the `message_id`. We can easily filter out all records for a particular push message from their `message_uuid` like:

```python
notifications = PushNotification.query.filter(and_(PushNotification.message_uuid == user_data['uuid'], PushNotification.sent == True))

user_list = [notification.chatID for notification in notifications]
message_ids = [notification.message_id for notification in notifications]
```

This provides us with lists of `chat_id`s and `message_id`s, which we can pipe to the `delete_threaded()` function:

```python
time_taken = delete_threaded(message_ids, user_list)
```

The function returns the time taken to send all delete requests to the Bot API, which can be useful for stats.

Note that we're only targeting users who actually received the message (`PushNotification.sent == True`).

## The Complete Implementation
I've only given a glimpse of my implementation, highlighting the key points required to build a push notification system for your bot. You can see my full implementation in these files:

- [misbot/admin.py](https://github.com/ArionMiles/MIS-Bot/blob/master/mis_bot/misbot/admin.py#L19)
- [misbot/push_notifications.py](https://github.com/ArionMiles/MIS-Bot/blob/master/mis_bot/misbot/push_notifications.py)

Below is my implementation in action.

![Screenshot](/images/push_notification_demo.jpg "Push Notification Screenshot")

As you may have noticed, it says I've sent the message to 702 users, but after deleting the notification it says only 687 messages have been deleted. The reason for this is that there were some inactive users, i.e. they may have blocked the bot or deleted their Telegram account. And, I delete all inactive users from the database after I'm done sending notifications.

The count of users generated before sending the notification includes inactive members too, but while deleting the notification, only those who were actually sent the message are targeted, hence the lower number.