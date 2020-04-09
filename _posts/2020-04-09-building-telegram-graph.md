---
layout: post
title:  "Building graph for Telegram chats, channels and their neighbors"
tags: graph telegram python plantuml linux cli shell
author: "github:freefd"
---

To be honest, networks are essentially can be represented as graphs, almost any relationships can be represented as graph. Do not also forget that network theory is a part of graph theory where nodes have attributes.

Today we will try to understand whether we are in information bubble created by the same authors of different channels, building a graph for visualization.

## Tools
In our project will be used [telethon](https://docs.telethon.dev/en/latest/) <sup id="a1">[1](#f1)</sup> Python 3 module for Telegram interaction, [NetworkX](https://networkx.github.io/) <sup id="a2">[2](#f2)</sup> for graph and [Plantuml](https://plantuml.com/) <sup id="a3">[3](#f3)</sup> for visualization.

## Building the graph
First thing we need is to understand which type of graph we must use: NetworkX provides several such [types](https://networkx.github.io/documentation/stable/reference/classes/index.html) <sup id="a4">[4](#f4)</sup>.

| NetworkX class | Type | Self-loops<br/>allowed | Parallel<br/>edges allowed |
| - | - | :-: | :-: |
| Graph | undirected | &#x2714;&#xFE0F; | &#x1f6ab; |
| DiGraph | directed | &#x2714;&#xFE0F; | &#x1f6ab; |
| MultiGraph | undirected | &#x2714;&#xFE0F; | &#x2714;&#xFE0F; |
| MultiDiGraph | directed | &#x2714;&#xFE0F; | &#x2714;&#xFE0F; |

The [quote](https://en.wikipedia.org/wiki/Loop_(graph_theory)) <sup id="a5">[5](#f5)</sup> from Wikipedia about self-loops:
> In graph theory, a **loop** (also called a **self-loop** or a "buckle") is an edge that connects a vertex to itself.

![self-looped graph](/images/2020-04-09-building-telegram-graph-01.png)

The [quote](https://en.wikipedia.org/wiki/Multiple_edges) <sup id="a6">[6](#f6)</sup> from Wikipedia about parallel edges:
> In graph theory, multiple edges (also called **parallel edges** or a **multi-edge**), are two or more edges that are incident to the same two vertices.

![multi-edged graph](/images/2020-04-09-building-telegram-graph-02.png)

Apparently, the beginning of our graph will be a user itself (let's call it "root node") and after that graph will be built around this node. No parallel edges are expected and graph could be unidirected, thus we can pick a simple Graph type.

To make our graph bigger, I'll subscribe/join several random channels grouped by topic found on [tgstat.com](https://tgstat.com/) <sup id="a7">[7](#f7)</sup>. But the main idea is to connect channels and chats to each other by common contacts which we can find in channel's and chat's information. For example, here is the Reddit channel information:

![Reddit channel info](/images/2020-04-09-building-telegram-graph-03.png)

Usernames starting with _@_ will represent other nodes on our graph and they can be connected with multiple channels and chats. As a result, we got a similar graph:

![graph example](/images/2020-04-09-building-telegram-graph-04.png)

## Writing the code

Before run the following code, we need to get [API ID and API hash](https://docs.telethon.dev/en/latest/basic/signing-in.html) <sup id="a8">[8](#f8)</sup>. Start from our root node named by _first name_ from Telegram client's profile with defined attribute _color_:

```python
#!/usr/bin/env python3

from telethon import TelegramClient, sync
import os
import networkx as nx

# Main configuration
config = {
    'telegram': {
        'api_id': '123456', # Client's API ID 
        'api_hash': 'bdaf62f128aeaa9a65b67a479d9ff413', # Client's API hash
        'phone_id': '+31201234567' # Client's phone
    }
}

if __name__ == '__main__':

    # Create Client and sign in
    client = TelegramClient(os.path.basename(__file__),
                            config['telegram']['api_id'],
                            config['telegram']['api_hash'])
    # Create Graph
    graph = nx.Graph()

    # Start Telegram client
    client.start(config['telegram']['phone_id'])
    client_name = client.get_me().first_name

    # Add Client as root node
    graph.add_node(client_name, color='gold')

    # Print all nodes from graph with their attributes
    print(graph.nodes(data=True))
```

Executing it:
```bash
~> python3 telegram_graph.py
[('Username', {'color': 'gold'})]
```

Now is the time to collect all the chats and channels for our client. We could do that using [client.get_dialogs()](https://telethonn.readthedocs.io/en/latest/extra/basic/entities.html) <sup id="a9">[9](#f9)</sup> function. Below I'm filtering the items only to _Reddit_ channel as an example:

```python
#!/usr/bin/env python3

from telethon import TelegramClient, sync
import os
import networkx as nx

# Main configuration
config = {
    'telegram': {
        'api_id': '123456', # Client's API ID 
        'api_hash': 'bdaf62f128aeaa9a65b67a479d9ff413', # Client's API hash
        'phone_id': '+31201234567' # Client's phone
    }
}

if __name__ == '__main__':

    # Create Client and sign in
    client = TelegramClient(os.path.basename(__file__),
                            config['telegram']['api_id'],
                            config['telegram']['api_hash'])
    # Create Graph
    graph = nx.Graph()

    # Start Telegram client
    client.start(config['telegram']['phone_id'])
    client_name = client.get_me().first_name

    # Get all dialogs for current client and filter them on the Reddit channel
    reddit_channel = [dialog for dialog in client.get_dialogs() if dialog.name == 'Reddit']
    
    # Add Client as root node
    graph.add_node(client_name, color='gold')

    # Print all nodes from graph with their attributes
    print(graph.nodes(data=True))

    # Print reddit_channel list containing only one element
    print(*reddit_channel)
```

Output:
```
[('Username', {'color': 'gold'})]
Dialog(name='Reddit', date=datetime.datetime(2020, 4, 8, 19, 0, 1, tzinfo=datetime.timezone.utc), draft=<telethon.tl.custom.draft.Draft object at 0x7f7ebcc0dcd0>, message=Message(id=8856, to_id=PeerChannel(channel_id=1236920376), date=datetime.datetime(2020, 4, 8, 19, 0, 1, tzinfo=datetime.timezone.utc), message='r/ #funny\nСпортзал закрыт? Импровизируй, адаптируйся!', out=False, mentioned=False, media_unread=False, silent=True, post=True, from_scheduled=False, legacy=False, from_id=None, fwd_from=None, via_bot_id=None, reply_to_msg_id=None, media=MessageMediaDocument(document=Document(id=5220052781397706607, access_hash=5929123456789142977, file_reference=b'\x02I\xb9\xe88\x00\x00"\x98^\x8e3j\xc7F\x00iQ\xdc\xe6\xeb~\x02nR\xf2\xde\xf0(', date=datetime.datetime(2020, 4, 8, 18, 0, 27, tzinfo=datetime.timezone.utc), mime_type='video/mp4', size=666658, dc_id=2, attributes=[DocumentAttributeVideo(duration=11, w=640, h=800, round_message=False, supports_streaming=True), DocumentAttributeFilename(file_name='r_gifs.mp4'), DocumentAttributeAnimated()], thumbs=[PhotoStrippedSize(type='i', bytes=b'\x01( xC\x96\xe3\xbd5\xd7\x08\xdcv\xab\x8a~f\xfa\xd3.?\xd51>\xdf\xce\x86\x08\xca\x05\xb7\xe0\x83\x8f\xa5H\x05-*\xf4\xac\xca!\x8e\xf6v\x97\x1b\xfe\xf5K$\xb2I\x13\x06rT\xf5\xaa\x11\x1d\xb2\x02jl\x85b\xbb\xb0\xc4\xfeub\x13\xccd\x18\x1d)\xbel\xa4`\x90A\xf4\xa6\xf9\xb9\xedMn{\x0f\xc2\x80\x1e",\xa1\x87z\x9bnB\x96_\x99z\x11\xde\x8a*[-"%\x8b#\x9a_$\x03\xedE\x15<\xcc,\x7f'), PhotoSize(type='m', location=FileLocationToBeDeprecated(volume_id=200020400599, local_id=30505), w=256, h=320, size=15312)]), ttl_seconds=None), reply_markup=None, entities=[MessageEntityHashtag(offset=3, length=6)], views=25322, edit_date=None, post_author=None, grouped_id=None), entity=Channel(id=1236920376, title='Reddit', photo=ChatPhoto(photo_small=FileLocationToBeDeprecated(volume_id=247538747, local_id=28490), photo_big=FileLocationToBeDeprecated(volume_id=247538747, local_id=28492), dc_id=2), date=datetime.datetime(2019, 8, 7, 19, 24, 51, tzinfo=datetime.timezone.utc), version=0, creator=False, left=False, broadcast=True, verified=False, megagroup=False, restricted=False, signatures=False, min=False, scam=False, has_link=False, has_geo=False, access_hash=643357336673217314, username='Reddit', restriction_reason=None, admin_rights=None, banned_rights=None, default_banned_rights=None, participants_count=214146))
```

But wait, there is nothing in the output about the contacts we saw above. Another method [GetFullChannelRequest](https://tl.telethon.dev/methods/channels/get_full_channel.html) <sup id="a10">[10](#f10)</sup> can help us solve this:

```python
#!/usr/bin/env python3

from telethon import TelegramClient, sync
from telethon.tl.types import Channel
from telethon.tl.functions.channels import GetFullChannelRequest
import os
import networkx as nx

# Main configuration
config = {
    'telegram': {
        'api_id': '123456', # Client's API ID 
        'api_hash': 'bdaf62f128aeaa9a65b67a479d9ff413', # Client's API hash
        'phone_id': '+31201234567' # Client's phone
    }
}

if __name__ == '__main__':

    # Create Client and sign in
    client = TelegramClient(os.path.basename(__file__),
                            config['telegram']['api_id'],
                            config['telegram']['api_hash'])
    # Create Graph
    graph = nx.Graph()

    # Start Telegram client
    client.start(config['telegram']['phone_id'])
    client_name = client.get_me().first_name

    # Get all dialogs for current client and filter them on the Reddit channel, get its full info
    reddit_channel = [dialog for dialog in client.get_dialogs() if dialog.name == 'Reddit']
    reddit_channel_fullinfo = client(GetFullChannelRequest(channel=reddit_channel[0]))

    # Add Client as root node
    graph.add_node(client_name, color='gold')

    # Print all nodes from graph with their attributes
    print(graph.nodes(data=True))

    # Print reddit_channel_fullinfo
    print(reddit_channel_fullinfo)
```

Output:
```
[('Username', {'color': 'gold'})]
ChatFull(full_chat=ChannelFull(id=1236920376, about='По рекламе @lulzzsecurity \nПрислать новость @redditgbot', read_inbox_max_id=8857, read_outbox_max_id=0, unread_count=0, chat_photo=Photo(id=515918679006882259, access_hash=4998218436240959217, file_reference=b'\x00^\x8eP\x91\xce\xd7jl\xe5f\xb7\xa7H\x05\x0b\x01\xea9\x88#', date=datetime.datetime(2019, 12, 21, 13, 31, 49, tzinfo=datetime.timezone.utc), sizes=[PhotoSize(type='a', location=FileLocationToBeDeprecated(volume_id=247538747, local_id=28490), w=160, h=160, size=10863), PhotoSize(type='b', location=FileLocationToBeDeprecated(volume_id=247538747, local_id=28491), w=320, h=320, size=25424), PhotoSize(type='c', location=FileLocationToBeDeprecated(volume_id=247538747, local_id=28492), w=640, h=640, size=52851)], dc_id=2, has_stickers=False), notify_settings=PeerNotifySettings(show_previews=None, silent=None, mute_until=datetime.datetime(2038, 1, 19, 3, 14, 7, tzinfo=datetime.timezone.utc), sound=None), exported_invite=ChatInviteEmpty(), bot_info=[], pts=44738, can_view_participants=False, can_set_username=False, can_set_stickers=False, hidden_prehistory=False, can_view_stats=False, can_set_location=False, participants_count=214142, admins_count=None, kicked_count=None, banned_count=None, online_count=None, migrated_from_chat_id=None, migrated_from_max_id=None, pinned_msg_id=None, stickerset=None, available_min_id=None, folder_id=1, linked_chat_id=None, location=None), chats=[Channel(id=1236920376, title='Reddit', photo=ChatPhoto(photo_small=FileLocationToBeDeprecated(volume_id=247538747, local_id=28490), photo_big=FileLocationToBeDeprecated(volume_id=247538747, local_id=28492), dc_id=2), date=datetime.datetime(2019, 8, 7, 19, 24, 51, tzinfo=datetime.timezone.utc), version=0, creator=False, left=False, broadcast=True, verified=False, megagroup=False, restricted=False, signatures=False, min=False, scam=False, has_link=False, has_geo=False, access_hash=643357336673217314, username='Reddit', restriction_reason=None, admin_rights=None, banned_rights=None, default_banned_rights=None, participants_count=None)], users=[])
```

Now the contacts are in place, we are able to create new nodes from chats, channels and contacts from full info, and then link them by edges. Every node type will have its own _color_ attribute (you remember that, right?):

```python
#!/usr/bin/env python3

from telethon import TelegramClient, sync
from telethon.tl.types import Channel
from telethon.tl.functions.channels import GetFullChannelRequest
from textwrap import wrap
import os
import re
import networkx as nx

# Main configuration
config = {
    'telegram': {
        'api_id': '123456', # Client's API ID 
        'api_hash': 'bdaf62f128aeaa9a65b67a479d9ff413', # Client's API hash
        'phone_id': '+31201234567' # Client's phone
    }
}

if __name__ == '__main__':

    # Create Client object and sign in
    client = TelegramClient(os.path.basename(__file__),
                            config['telegram']['api_id'],
                            config['telegram']['api_hash'])
    # Create Graph object
    graph = nx.Graph()

    client.start(config['telegram']['phone_id'])
    client_name = client.get_me().first_name

    # Add Client as root node
    graph.add_node(client_name, color='gold')#config['graph']['color']['client'])


    # For each not ignored channel in list of dialogs
    for channel in [dialog.entity for dialog in client.get_dialogs()
                        if isinstance(dialog.entity, Channel) and 
                           dialog.entity.id not in config['graph']['channels_ignore']]:

        # Get full information for a channel and word wrap its name
        channel_full_info = client(GetFullChannelRequest(channel=channel))
        channel_name = '\\n'.join(wrap(channel.title, config['graph']['wordwrap_length']))

        # Add channel ID as node with attributes 'title' and 'color', link it to the root node
        graph.add_node(channel.id, title=channel_name, color=config['graph']['color']['channel'])
        graph.add_edge(client_name, channel.id)

        # For each contact in full information 
        for contact_name in re.findall("@([A-z0-9_]{1,100})", channel_full_info.full_chat.about):

            # Add contact as node with attribute and link to the channel node
            graph.add_node(contact_name, color=config['graph']['color']['user'])
            graph.add_edge(contact_name, channel.id) 

    # Print all nodes from graph with their attributes
    print(graph.nodes(data=True))
```


## Final script

Below you can find the [final script](https://github.com/freefd/utils/blob/master/telegram_graph.py) <sup id="a11">[11](#f11)</sup> that build a Plantuml file.

```python
#!/usr/bin/env python3

from telethon import TelegramClient, sync
from telethon.tl.types import Channel
from telethon.tl.functions.channels import GetFullChannelRequest
from textwrap import wrap
import os
import re
import networkx as nx

# Main configuration
config = {
    'telegram': {
        'api_id': '123456', # Client's API ID 
        'api_hash': 'bdaf62f128aeaa9a65b67a479d9ff413', # Client's API hash
        'phone_id': '+31201234567' # Client's phone
    },
    'graph': {
        'channels_ignore': [], # Channels to ignore
        'color': { # Colors for nodes (https://plantuml.com/en/color for more information) [12]
            'client': 'gold',
            'channel': 'technology',
            'user': 'lavender'
        },
        'title': 'Telegram channels relationships', # Graph title
        'wordwrap_length': 15
    }
}

if __name__ == '__main__':

    # Create Client object and sign in
    client = TelegramClient(os.path.basename(__file__),
                            config['telegram']['api_id'],
                            config['telegram']['api_hash'])
    # Create Graph object
    graph = nx.Graph()

    client.start(config['telegram']['phone_id'])
    client_name = client.get_me().first_name

    # Add Client as root node
    graph.add_node(client_name, color=config['graph']['color']['client'])

    # For each not ignored channel in list of dialogs
    for channel in [dialog.entity for dialog in client.get_dialogs()
                        if isinstance(dialog.entity, Channel) and 
                           dialog.entity.id not in config['graph']['channels_ignore']]:

        # Get full information for a channel and word wrap its name
        channel_full_info = client(GetFullChannelRequest(channel=channel))
        channel_name = '\\n'.join(wrap(channel.title, config['graph']['wordwrap_length']))

        # Add channel ID as node with attributes 'title' and 'color', link it to the root node
        graph.add_node(channel.id, title=channel_name, color=config['graph']['color']['channel'])
        graph.add_edge(client_name, channel.id)

        # For each contact in full information 
        for contact_name in re.findall("@([A-z0-9_]{1,100})", channel_full_info.full_chat.about):

            # Add contact as node with attribute and link to the channel node
            graph.add_node(contact_name, color=config['graph']['color']['user'])
            graph.add_edge(contact_name, channel.id) 

    # Create Planutml file object
    plantumlFile = open("{}_telegram_graph.plantuml".format(client_name), 'w')

    # Write Plantuml header with graph title
    plantumlFile.write("@startuml\ntitle {}\nleft to right direction\n".format(config['graph']['title']))

    # For each node in graph
    for node in graph.nodes(data=True):

        # The node is channel if it has 'title' attribute
        if 'title' in node[1]:
            plantumlFile.write('frame {} as "{}" #{}\n'.format(node[0], node[1]['title'], node[1]['color']))

        # Otherwise, the node is contact
        else:
            plantumlFile.write('usecase {0} as "@{0}" #{1}\n'.format(node[0], node[1]['color']))

    # Link the nodes with each other by edges
    for edge in graph.edges():
        plantumlFile.write('{} 0--# {}\n'.format(edge[0], edge[1]))
    
    # Write Plantuml footer and close the file
    plantumlFile.write("@enduml")
    plantumlFile.close()
```

After completion of execution it will create a file named by _first name_ from Telegram client's profile concatenated with __telegram_graph.plantuml_. Example content:

```java
@startuml
title Telegram channels relationships
left to right direction
usecase Username as "@Username" #gold
frame 1035713458 as "ntwrk" #technology
usecase zhenyatsk as "@zhenyatsk" #lavender
usecase mxssl as "@mxssl" #lavender
usecase darwinggl as "@darwinggl" #lavender
frame 1280552026 as "Коронавирус\nLIVE" #technology
usecase pr_virus as "@pr_virus" #lavender
usecase virusologbot as "@virusologbot" #lavender
frame 1378813139 as "Baza" #technology
usecase mogutin as "@mogutin" #lavender
usecase bazanewsbot as "@bazanewsbot" #lavender
frame 1236920376 as "Reddit" #technology
usecase lulzzsecurity as "@lulzzsecurity" #lavender
usecase redditgbot as "@redditgbot" #lavender
...
Username 0--# 1035713458
Username 0--# 1280552026
Username 0--# 1378813139
Username 0--# 1236920376
...
1035713458 0--# zhenyatsk
1035713458 0--# mxssl
1035713458 0--# darwinggl
1280552026 0--# pr_virus
1280552026 0--# virusologbot
pr_virus 0--# 1470273900
virusologbot 0--# 1470273900
1378813139 0--# mogutin
1378813139 0--# bazanewsbot
1236920376 0--# lulzzsecurity
1236920376 0--# redditgbot
...
@enduml
```

## Graph visualization
To tell the truth, in the beginning I wanted to make a visualization using classic [matplotlib.pyplot](https://matplotlib.org/api/pyplot_api.html) <sup id="a13">[13](#f13)</sup> library. But after a several tries to draw a well formed graph, I changed my mind and pick [Plantuml](https://plantuml.com/) <sup id="a3">[3](#f3)</sup> for visualization. It supports many topologies and cases, plenty image types as PNG, SVG and LaTeX for a graph output.

PlantUML limits image width and height to 4096, but there is PLANTUML_LIMIT_SIZE environment variable that we can set to override this limit during launch of Plantuml. Let's draw our graph we got at the previous chapter, by default Plantuml generates PNG image:

```bash
~> ls -1
telegram_graph.py

~> python3 telegram_graph.py
Please enter the code you received: 12345
Signed in successfully as Username

~> ls -1
Username_telegram_graph.plantuml
telegram_graph.py
telegram_graph.py.session

~> java -DPLANTUML_LIMIT_SIZE=8192 -Djava.awt.headless=true -jar /path/to/plantuml.jar ./Username_telegram_graph.plantuml

~> ls -1
Username_telegram_graph.plantuml
Username_telegram_graph.png
telegram_graph.py
telegram_graph.py.session
```
Now you can open _Username_telegram_graph.png_ with your favorite image viewer to inspect how many channels have link to same contacts.
As for me, there were few such channels:

![Username graph](/images/2020-04-09-building-telegram-graph-05.png)

A little bit close-up the one piece:

![Part of username graph](/images/2020-04-09-building-telegram-graph-06.png)

## Bonus: CLI one-liner

Following the previous [article](/2019/11/03/old-fashioned-way-about-data-science.html) we already learned that write code is not always necessary. Some cases could be solved in CLI only: [curl](https://curl.haxx.se/) <sup id="a14">[14](#f14)</sup> and [jq](https://stedolan.github.io/jq/) <sup id="a15">[15](#f15)</sup> do magic. For example, let us investigate how many channels use [Social Energy](https://segroup.me/en/) <sup id="a16">[16](#f16)</sup> agency (actually we also can do this [here](https://tg.segroup.me/en) <sup id="a17">[17](#f17)</sup>):

```bash
$ ( echo -e '@startuml\nleft to right direction\nusecase social_energy as "@social_energy" #lavender'; curl -sX POST https://tgstat.com/channels/list --data 'offset=0&period=yesterday&country=global&language=global&verified=0&price[vp]=0&search=@social_energy' | jq '.items.list[] | "frame \(.id) as \"\(.title)\" #technology\nsocial_energy -->> \(.id)"' -r | sort -V; echo "@enduml" ) | java -DPLANTUML_LIMIT_SIZE=8192 -jar /path/to/plantuml.jar -pipe > social_energy_telegram_graph.png
```

Explaining step by step. Create a Plantuml header and an actor:
```bash
$ echo -e '@startuml\nleft to right direction\nusecase social_energy as "@social_energy" #lavender';
@startuml
left to right direction
usecase social_energy as "@social_energy" #lavender
```
Request a list of channels from [tgstat.com](https://tgstat.com) <sup id="a7">[7](#f7)</sup> for _@Social_Energy_ contact and sort output lines:
```bash
curl -sX POST https://tgstat.com/channels/list --data 'offset=0&period=yesterday&country=global&language=global&verified=0&price[vp]=0&search=@social_energy' | jq '.items.list[] | "frame \(.id) as \"\(.title)\" #technology\nsocial_energy -->> \(.id)"' -r | sort -V
frame 55002 as "Женский Гороскоп" #technology
frame 56218 as "Интересные факты" #technology
frame 62891 as "Факты | Наука | Фильмы" #technology
...
social_energy -->> 55002
social_energy -->> 56218
social_energy -->> 62891
...
```
Print Plantuml footer:
```bash
$ echo "@enduml"
@enduml
```
All this text is pipelined to Plantuml with _-pipe_ option and the result will be written to [_social_energy_telegram_graph.png_](/images/2020-04-09-building-telegram-graph-07.png) file.


## References
<b id="f1">1</b>. [Python3 telethon library](https://docs.telethon.dev/en/latest/) [↩](#a1)<br/>
<b id="f2">2</b>. [Python NetworkX library for complex networks](https://networkx.github.io/) [↩](#a2)<br/>
<b id="f3">3</b>. [PlantUML tool allowing create diagrams from a plain text language](https://plantuml.com/) [↩](#a3)<br/>
<b id="f4">4</b>. [NetworkX graph types](https://networkx.github.io/documentation/stable/reference/classes/index.html) [↩](#a4)<br/>
<b id="f5">5</b>. [Loop meaning in graph theory](https://en.wikipedia.org/wiki/Loop_(graph_theory)) [↩](#a5)<br/>
<b id="f6">6</b>. [Multi-edge meaning in graph theory](https://en.wikipedia.org/wiki/Multiple_edges) [↩](#a6)<br/>
<b id="f7">7</b>. [Telegram Analytics](https://tgstat.com/) [↩](#a7)<br/>
<b id="f8">8</b>. [Telegram API ID and API Hash](https://docs.telethon.dev/en/latest/basic/signing-in.html) [↩](#a8)<br/>
<b id="f9">9</b>. [Work with users, chats and channels
in telethon](https://telethonn.readthedocs.io/en/latest/extra/basic/entities.html) [↩](#a9)<br/>
<b id="f10">10</b>. [Get full channel info in telethon](https://tl.telethon.dev/methods/channels/get_full_channel.html) [↩](#a10)<br/>
<b id="f11">11</b>. [telegram_graph.py at Github](https://github.com/freefd/utils/blob/master/telegram_graph.py) [↩](#a11)<br/>
<b id="f12">12</b>. [Plantuml supported colors list](https://plantuml.com/en/color)<br/>
<b id="f13">13</b>. [Python Matplotlib library](https://matplotlib.org/api/pyplot_api.html) [↩](#a13)<br/>
<b id="f14">14</b>. [Command line tool for transferring data with URLs](https://curl.haxx.se/) [↩](#a14)<br/>
<b id="f15">15</b>. [Lightweight and flexible command-line JSON processor](https://stedolan.github.io/jq/) [↩](#a15)<br/>
<b id="f16">16</b>. [Social Energy platform](https://segroup.me/en/) [↩](#a16)<br/>
<b id="f17">17</b>. [Social Enegry prices](https://tg.segroup.me/en) [↩](#a17)<br/>
