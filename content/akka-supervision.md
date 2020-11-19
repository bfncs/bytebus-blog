+++
title = "Supervision and watching child actors in Akka"
date = 2020-11-19
+++

## Overview

[Akka Supervision](https://doc.akka.io/docs/akka/current/general/supervision.html) is a very powerful tool to build a fault tolerant Akka application. In this post we'll explore how to use it to handle failure of an actor and how to watch child actors. 

## Scenario

To be able to play around and understand supervision, let's quickly create a test scenario, consisting of:

1) an **actor system**, with
2) a **parent actor**, that acts as user guardian and doesn't do anything but start
3) a **child actor**, that behaves suicidal by sending itself a message causing an unhandled RuntimeException one second after starting.

Sounds way more dramatic than it is, so let's look at the code:

```java
public final class App {

  public static void main(String[] args) {
    ActorSystem.create(createParent(), "supervision");
  }

  private static Behavior<Object> createParent() {
    return Behaviors.setup(
        ctx -> {
          ctx.spawn(createChild(), "child");
          return Behaviors.empty();
        });
  }

  private static Behavior<String> createChild() {
    return Behaviors.withTimers(
        timers -> {
          timers.startSingleTimer("illegal", Duration.ofSeconds(1));
          return Behaviors.receive(
              (ctx, msg) -> {
                ctx.getLog().info("Child received msg: {}", msg);
                throw new IllegalStateException("illegal message");
              });
        });
  }
}
```

When running this example, the application dies after the first occurrence of the exception.

## Supervision

Suppose the child actor does something actually useful, builds up state and then runs into a failure. The Akka way to handle this would be to *let it crash* and restart the actor through a [supervision strategy](https://doc.akka.io/docs/akka/current/typed/fault-tolerance.html#supervision), so it can continue working with a valid state.

The default supervision strategy is to just stop the actor. To instead restart the actor no more than 3 times in a 10 second period, just wrap your behavior in `Behaviors.supervise()` with the according supervision strategy:
```java
  private static Behavior<String> createChild() {
    return Behaviors.<String>supervise(
            Behaviors.withTimers(
                timers -> {
                  timers.startSingleTimer("illegal", Duration.ofSeconds(1));
                  return Behaviors.receive(
                      (ctx, msg) -> {
                        ctx.getLog().info("Child received msg: {}", msg);
                        throw new IllegalStateException("illegal message");
                      });
                }))
        .onFailure(SupervisorStrategy.restart().withLimit(3, Duration.ofSeconds(10)));
  }
```

When running the updated example, the application only dies after (re-)starting the actor (and running processing until the exception is thrown) for four times.

It's possible to handle different exceptions with different strategies and to use more complex strategies, e.g. restart with backoff.

## Watching child actor

Suppose the parent actor cannot do its work, when the child actor is failing permanently, even after possible restarts. In that case we probably want the parent itself to die with the child (and possibly use a supervision strategy like above to recover).

To add this behavior, we have to `watch` the child actor:

```java
  private static Behavior<Object> createParent() {
    return Behaviors.setup(
        ctx -> {
            final ActorRef<String> child =
              ctx.spawn(createChild(), "child");
            ctx.watch(child);
            return Behaviors.empty();
        });
```

When running the updated application, after the child actor fails for the fourth time, the application fails with a `DeathPactException`:

```
akka.actor.typed.DeathPactException: death pact with Actor[akka://supervision/user/child#-1634369735] was triggered
```

We see this because the parent is now watching the child actor, so if the child is stopped because of a failure, the parent will receive a [`ChildFailed`](https://doc.akka.io/api/akka/current/akka/actor/typed/ChildFailed.html) signal – and will itself fail, if the signal is not handled.

If you need to do something more involved when the child fails, you should add custom behavior on receiving a signal:

```java
  private static Behavior<Object> createParent() {
    return Behaviors.setup(
        ctx -> {
          final ActorRef<String> childRef = ctx.spawn(createChild(), "child");
          ctx.watch(childRef);
          return Behaviors.receiveSignal(
              (context, signal) -> {
                if (signal instanceof ChildFailed) {
                  ctx.getLog().error("Child failed: {}", signal);
                  // handle child failure here 
                }
                return Behaviors.same();
              });
        });
  }
```

When running the updated example, the application will not die any more but just log the child failure.

## Binding Akka streams pipelines

If you are using Akka Streams in your application, it is good to know that it uses a [very similar take on supervision](https://doc.akka.io/docs/akka/current/stream/stream-error.html#supervision-strategies) to handle failure within the stream.

If you want to stop and possibly restart a stream, when an actor fails, you will have to [bind it to the actor's lifecycle](https://doc.akka.io/docs/akka/current/stream/stream-flows-and-basics.html#actor-materializer-lifecycle) through the a custom instance of `Materializer`.

Let's try this with a stream that is just counting up and logging.

```java

  private static Behavior<String> createChild() {
    return Behaviors.<String>supervise(
            Behaviors.setup(
                ctx -> {
                    final Materializer materializer = Materializer.createMaterializer(ctx);
                    Source.range(0, 100)
                      .throttle(1, Duration.ofMillis(200))
                      .log("processed msg")
                      .to(Sink.ignore())
                      .withAttributes(
                          ActorAttributes.logLevels(
                              Logging.InfoLevel(), Logging.InfoLevel(), Logging.ErrorLevel()))
                      .run(materializer);

                  return Behaviors.withTimers(
                      timers -> {
                        timers.startSingleTimer("illegal", Duration.ofSeconds(1));
                        return Behaviors.receive(
                            (context, msg) -> {
                              ctx.getLog().info("Child received msg: {}", msg);
                              throw new IllegalStateException("illegal message");
                            });
                      });
                }))
        .onFailure(SupervisorStrategy.restart().withLimit(3, Duration.ofSeconds(10)));
  }
```

When running this updated example, you can see that every time after (re-)starting the children, the stream is starting to count up from `0`.
