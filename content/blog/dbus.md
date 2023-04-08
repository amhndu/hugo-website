+++
title = "Connect and control your media player with python and dbus using MPRIS"
date = "2015-06-15"
+++

In this guide we'll be using Python 3 although you may be able to apply this to Python 2.4 and above with minor adjustments. Since D-Bus is only used by Linux and BSD distributions, needless to say, this will only work with \*nix systems

We'll connect to a media player which implements the **Media Player Remote Interfacing Specification** or MPRIS for short. Most popular players like Audacious, Amarok, Clementine, VLC etc will work.
This can be useful for purposes like priting metadata of the current track on [conky](../Projects/conky-cards.html) and controling the player with widgets. For a particular example, Clementine, for some reason would not bind the media keys on my keyboard for global shortcuts, so with the help of a python script I made the keybindings directly with the window manager and my script.

This guide follows the python documentation convention for code.
`>>>` imples the python interpreter prompt, ... imples a multiline prompt continuing from the previous command and a line with neither of the prompts implies output, therefore, if you are copy-pasting code, you will have to copy a line at a time. Also, you should be versed in python terminology and should have some basic knowledge of Python.
Let's get started, launch a terminal with the python interpreter and put on some music!

### D-Bus Basics

DBus is a message bus system which allows _Inter-process communication_ i.e it allows different processes to talk to each other and even different instances of the same application.
The _dbus daemon_ runs in the background allowing applications to connect to this _daemon_ using the _libdbus_ library and it's wrappers for different languages and frameworks.
Communication is done using a **bus**, each application is granted a unique name like _:1-45_ or _:2-16_) when it connects to the bus. The application may request a familiar bus name (based on the [Reversed domain name](https://en.wikipedia.org/wiki/Reverse_domain_name_notation) convention) e.g. _org.freedesktop.PowerManagement, com.Skype.API and org.mpris.MediaPlayer2.clementine_
Each application or service can have multiple **object paths** representing a unique object within the application, they look like filesystem paths e.g. _/org/mpris/MediaPlayer2_
**Interfaces** act like namespaces, within which **Methods** and **Signals** are defined.
Therefore, to make a method call you need:

*   Bus name
*   Object path
*   Interface name

Some applications may also have **Properties**, these properties could be set or retrieved through the _org.freedesktop.DBus.Properties_ interface. More on this later.
There are mainly two two buses, the **Session Bus** and the **System Bus** The system bus is a session independent bus which is used by system services such as _udev_ or _NetworkManager_ whereas a session bus is unique for the current user session. It is used by desktop applications and it is this bus we will be interested in for this guide.

### Connecting to the D-Bus daemon with Python

    >>> import dbus
    >>> bus = dbus.SessionBus()

In the first line, we import the dbus module and in the second line we make an object of the class **SessionBus**.
This class will make us a connection with the session bus granting us a unique name which can be retrieved with the get\_unique\_name() method. Try this

    >>> bus.get_unique_name()
    ':1.113'

Your unique name will vary.
We can also request a name with the method request\_name() following the reverse domain convention.

    >>> bus.request_name('io.github.amhndu.test')

A return value of 1 implies that you have succesfully acquired the name.
This class also allows us to list all the services/applications registered with the session bus:

    >>> for service in bus.list_names():
    ...     print(service)

In the list you'll also see the name we requested above.
You can even expose your python application with an API on dbus with this module but it is beyond this article's scope.
Note: D-bus, unlike python, is strongly typed and the python dbus library has wrapper classes for datatypes to faciliate interfacing dbus methods and python, thus the methods in the dbus module return these wrapper classes which can be implicitly converted to python native data types.

### Connecting to the player

The [MPRIS 2.0](#mpris-spec) spec defines that the player must request a name prefixed with _org.mpris.MediaPlayer2_. Therefore a player can have a name of form _org.mpris.MediaPlayer2.clementine_ where the name ends the the name of the player.
To find out all the services of this form we can use regex to filter it out:

    >>> import re
    >>> for service in bus.list_names():
    ...     if re.match('org.mpris.MediaPlayer2.', service):
    ...             print(service)
    ...
    org.mpris.MediaPlayer2.clementine
    org.mpris.MediaPlayer2.vlc

As you can see in output, I had two players --vlc and clementine-- opened.
Once you've found the service name of your player, we can get an object:

    >>> player = dbus.SessionBus().get_object('org.mpris.MediaPlayer2.clementine', '/org/mpris/MediaPlayer2')

The first argument to get\_object() is the service name, while the second is the object path. The object path we've used is defined by [MPRIS](#mpris-spec).
Try printing these attributes of player -- bus\_name and requested\_bus\_name

### Making method calls

To make method calls, we call the method name directly on the object, giving it the associated interface name.
With [MPRIS](#mpris-spec), the interface _org.mpris.MediaPlayer2.Player_ defines methods like Next(), therefore to switch the track to the next in playlist, we call player.Next() and set the dbus\_interface keyword argument as the interface

    >>> player.Next(dbus_interface='org.mpris.MediaPlayer2.Player')

You should notice the track change.
Similarly we can call Previous(), Play(), Pause() or PlayPause()
But specifying the interface on each call is inconvenient, we can instead get an interface object once and rid ourselves of specifying it each time.

    >>> interface = dbus.Interface(player, dbus_interface='org.mpris.MediaPlayer2.Player')
    >>> interface.Next()
    >>> interface.Previous()
    >>> interface.Pause()
    >>> interface.PlayPause() #Play if paused, pause if playing

Very straightforward, isn't it ?

### Properties

We'll have to use the _org.freedesktop.DBus.Properties_ interface to get the property.
This interface has three methods -- Get(), Set(), GetAll() with which you can get or set the property in an interface.
Therefore to get a property in _org.mpris.MediaPlayer2.Player_ interface, we call Get() in _org.freedesktop.DBus.Properties_ with the name of the interface and our property to retrieve it and Set() to set it.
In python:

    >>> volume = player.Get('org.mpris.MediaPlayer2.Player', 'Volume',
    ...             dbus_interface='org.freedesktop.DBus.Properties')
    >>> print(volume)

We can of course make an interface object to faciliate making frequent calls.

    >>> property_interface = dbus.Interface(player, dbus_interface='org.freedesktop.DBus.Properties')
    >>> volume = property_interface.Get('org.mpris.MediaPlayer2.Player', 'Volume')
    >>> print(volume)
    0.9
    >>> property_interface.Set('org.mpris.MediaPlayer2.Player', 'Volume', volume-0.2)

Here, we make an interface object, retrieve the volume (which is a value between 0.0 to 1.0) and in the last line, change the volume.
The GetAll() method can be used print all the properties defined in an interface. e.g.

    >>> for property, value in property_interface.GetAll('org.mpris.MediaPlayer2.Player').items():
    ...     print(property, ':', value)

You can see all the properties defined along with the _Metadata_ dictionary.

    ### Metadata

A dictionary with _Metadata_ of the current track playing can be obtained as below.

    >>> metadata = player.Get('org.mpris.MediaPlayer2.Player', 'Metadata',
    ...             dbus_interface='org.freedesktop.DBus.Properties')

The peculiar thing about this dictionary is the naming system of the attributes. This is an example of it's contents:

    >>> for attr, value in metadata.items():
    ...     print(attr, '\t', value)
    ...
    xesam:contentCreated 	 2015-01-16T16:23:06
    xesam:album 	 Radioactive
    xesam:title 	 Radioactive
    mpris:trackid 	 /org/mpris/MediaPlayer2/Track/165
    xesam:useCount 	 2
    mpris:artUrl 	 file:///tmp/clementine-art-yK1220.jpg
    xesam:url 	 file:///home/amish/Music/Imagine Dragons/imagine-dragons---radioactive.mp3
    mpris:length 	 185000000
    xesam:trackNumber 	 1
    xesam:autoRating 	 38
    xesam:artist 	 dbus.Array([dbus.String('Imagine Dragons')], signature=dbus.Signature('s'), variant_level=1)
    xesam:genre 	 dbus.Array([dbus.String('Blues')], signature=dbus.Signature('s'), variant_level=1)
    bitrate 	 256
    xesam:lastUsed 	 2015-06-09T13:27:53

The attribute name follows the ["Xesam ontology"](http://www.freedesktop.org/wiki/Specifications/mpris-spec/metadata/). Note that the _xesam:artist_ and _xesam:genre_ attributes above are _dbus arrays_ which can used like python lists, these arrays will mostly have a singular item which can be accesed this way:

    >>> print('Artist :\t', metadata['artist'][0])
    Switchfoot

Note: Only the xesam:trackid is guaranteed to exist in the metadata dictionary, so, accessing attributes should be done with a check whether it exists.

    >>> print('Title :\t', metadata['xesam:title'] if 'xesam:title' in metadata else 'Unknown')

We used the ternary conditional operator to make it succint.
For more information on metadata, see the [MPRIS v2 metadata guidelines](http://www.freedesktop.org/wiki/Specifications/mpris-spec/metadata/)

### Conclusion

Now, you should be able to make your own scripts or just use the interactive interpreter to control your media player. You can have a look at [mediaplayer.py](https://github.com/amhndu/conky-cards/blob/master/mediaplayer.py), a python script I made which I use to print metadata in [conky](https://github.com/amhndu/conky-cards) and make keybindings for clementine, which I was unable to do through clementine itself. For more information, see the D-Bus and MPRIS specifications in the references section below.

### References

1.  Carlson, Herzberg, _et al._ "D-Bus Specification"
    [http://dbus.freedesktop.org/doc/dbus-specification.html](http://dbus.freedesktop.org/doc/dbus-specification.html)
2.  The VideoLAN team, _et al._ "MPRIS D-Bus Interface Specification"
    [http://specifications.freedesktop.org/mpris-spec/latest](http://specifications.freedesktop.org/mpris-spec/latest)
3.  McVittie, Simon "Dbus-python tutorial"
    [http://dbus.freedesktop.org/doc/dbus-python/doc/tutorial.html](http://dbus.freedesktop.org/doc/dbus-python/doc/tutorial.html)
