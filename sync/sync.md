# Synchronization(同步)

在执行了thread之后，下个要问的问题就是要怎么在thread之间去做synchronization(同步)。我们分以下几个问题来讨论:

1. [Resource Sharing](sync/resource_sharing.md): 如果多个threads同时存取变量该如何解決? 是否可以同时只有我这个thread允许修改某一个resource?
2. [Flow Control](sync/flow_control.md): 一个thread可否等到另外一个thread完成或是执行到某个状态的时候才继续往下执行?
3. [Message Passing](sync/message_passing.md): 我可否把我一个thread的output当作另外一个thread的input?
