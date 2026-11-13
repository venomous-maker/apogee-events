# apogee-events

Events for [Apogee](https://github.com/venomous-maker): something happened;
whoever cares is told. Typed listeners, listener classes built by the
container, subscribers, discovery, named events with wildcards, priorities,
stoppable events, dispatch held until a transaction commits, queued
listeners, async dispatch, and a fake for tests.

```grav
import "std"
import "apogee-events"

class OrderShipped extends StoppableEvent {
    public orderId: long
    constructor(id: long) { self.orderId = id }
}

class SendShipmentEmail extends Listener {
    private mailer: Mailer
    constructor(mailer: Mailer) { self.mailer = mailer }     // injected by the container
    fn handle(e: OrderShipped) { self.mailer.send(...) }
    fn shouldQueue() -> bool { return true }                  // run on a queue, not during dispatch
}

let events = new Events(container)
events.listen::<OrderShipped>(fn(e: OrderShipped) { audit(e.orderId) }, 10)   // priority 10
events.listenClass::<OrderShipped, SendShipmentEmail>()
events.discover()                                  // every Listener subclass, by its handle(e)'s type
events.on("user.*", fn(name: string, payload: any) { ... })
events.dispatch(new OrderShipped(42))
```

## Listening

| | |
|---|---|
| `listen::<E>(f, priority = 0)` | `f` hears every `E`, and every subclass of `E` |
| `once::<E>(f)` | the next one only |
| `listenClass::<E, L>()` | `L` is built by the container for each event (its constructor's dependencies injected) and its `handle(e: E)` called |
| `subscribe(s)` | a `Subscriber`'s `subscribe(events)` registers several |
| `discover()` | registers every concrete `Listener` subclass by the event type its `handle` takes (they all have reflection: `Listener`'s decorator); how many |
| `on(pattern, f, priority)` | a named event: `"user.login"`, `"user.*"`, `"*.created"`, `"*"` |
| `forget::<E>()`, `hasListeners::<E>()` | |

An event's listeners are its own class's and every base class's: by priority
(higher first), then the more specific class first, then the order they were
added. An event extending `StoppableEvent` can `stop()`: later listeners are
not told.

## Dispatching

- `dispatch(event)` tells them now and returns the event; `fire(name, payload)`
  a named event.
- `dispatchAsync(event)` runs it on the worker pool (a `Future`).
- `transaction(body)` holds the events dispatched inside `body` and
  dispatches them, in order, once it returns, or drops them when it throws:
  run it around a database transaction to tell listeners only about what was
  committed.
- A listener class whose `shouldQueue()` is true goes to the queue hook
  (`queueUsing(hook)`, which apogee-queue sets) with its `queue()` name,
  instead of running during dispatch.

## Testing

```grav
events.fake()                       // or fake(["app.OrderShipped"]): only those
checkout()
events.assertDispatched::<OrderShipped>(1)
events.assertNotDispatched::<OrderCancelled>()
events.dispatched::<OrderShipped>()  // the events themselves
```

## Speed

A class's full listener list (bases merged, ordered) is worked out once and
then read under a short lock; listeners run outside it. `--release`:
dispatching to three listeners takes about 190 ns.

## Tests

```bash
grav test tests/events.grav
```

MIT licensed.
