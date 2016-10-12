# Processes

> WARNING: This part needs some heavy thinking.

Unlike channels, processes are not addressable. They are simply a processing units, that will work until stopped.

Processes have few distinct traits:

- They never work on multiple threads.
- They never fail. Even thou underlying operation may fail, process itself will always survive. In case of failure it may simply be restarted.

Unlike actors, processes are not bound to a single channel. They may operate on many different channels.