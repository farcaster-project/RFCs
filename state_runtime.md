# Daemon protocol-state and runtime
## Information flow across components as string diagram
![](https://i.imgur.com/XuXia5t.png)



## Information flow as petrinet

```
client: Enab -> Inst.
daemon: Inst I Ev1 Prot2  Ev2 -> Enab I Job1 Prot1 Job2.
syncer1: Job1  B1 -> B1  Ev1.
syncer2: Job2  B2 -> B2 Ev2.
init: () -> Enab I Prot2 Ev1 Ev2 B1 B2.
counter_daemon: Prot1 -> Prot2.
```
In the petrinet below the red wires are input messages to the Daemon.  
![](https://i.imgur.com/nTTGGvr.png)


### Recover state
$L$ is the complete set of valid logs produced by valid protocol executions and $\Sigma$ is the set of protocol states.

There is a function, `recover_state` or $r$,

$$ L \rightarrow \Sigma $$
$$r(logs) = \sigma $$

 that takes the time-ordered logs composed of 
- events chain 1, $Ev1$
- events chain 2, $Ev2$
- client instructions, $Inst$
- counter-party protocol messages, $Prot2$
- self protocol messages, $Prot1$
- internal-timers, $I$ or $Loopback$
- published jobs, $Job1$
- published jobs, $Job2$

and that outputs a protocol state, $\sigma$.

### Update state
There is a function, `update_state` or $u$,
$$\Sigma\times{L} \rightarrow \Sigma$$
$$u(\sigma_{t-1}, \Delta logs) = \sigma_{t}$$

where $\sigma_{t-1}$ is the previous state and $\sigma_{t}$ is the final state, and $\Delta logs$ are the logs between t-1 and t.

Background on [petrinets](http://petrinet.org/) for the following.

The protocol state may be encoded as markings of a petrinet.

In practice the code may be implemented as a fold operation over a stream of events taking a petrinet marking as initial state that internally outputs the new marking---which encodes the new protocol state. Then tap the wire after the fold operation to monitor the state.

