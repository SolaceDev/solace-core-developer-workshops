# Publish and subscribe

Acme Air publishes flight, baggage and booking events. One subscriber, the
baggage service, listens for the baggage events and nothing else. The broker
delivers those and throws the other two away, because nobody asked for them.

Press **Play**.

## Anatomy of a topic

Every event is published to a complete, specific topic. Nothing is addressed
to a queue or to a consumer.

```
acme/air/flight/departed/v1/AC8763
 |    |     |        |     |    |
 org  domain object  action version key
```

Read it left to right, most constant to most variable. The convention used
throughout this workshop is:

```
{org}/{domain}/{object}/{action}/{version}/{context...}/{id}
```

Some rules that save pain later:

- **Past tense for events.** `departed`, not `depart`. An event states
  something that already happened.
- **Version in the middle.** A `v2` can be published alongside `v1` without
  disturbing anyone subscribed to `v1`.
- **Only what routes.** A field belongs in the topic if someone might filter
  on it. A trace ID does not.
- **No environment names.** `dev` and `prod` belong to different brokers, not
  different topic levels.

## Choosing a subscription

The baggage service subscribes to:

```
acme/air/baggage/*/v1/*
```

`*` matches exactly one level. So this reads as every baggage action, for
every flight, in version 1. It is the most specific subscription that still
catches every baggage event the app can read. You can see it on the line from
the broker to Baggage in the diagram.

`>` matches everything from that level down, so `acme/air/baggage/>` would
also work today. It would also deliver a `v2` event the app cannot parse, or
a topic with extra levels added later. Ask for what you can handle.

Watch the publisher and the subscriber side by side. Three events go to the
broker every round and one comes out. The flight and booking events match no
subscription, so the broker discards them. With direct messaging that is not
an error: nobody asked for them.

## The ACL is the ceiling

The baggage user's ACL profile allows `acme/air/baggage/>`. The app asks for
less than that, which is fine. The ACL decides what a client may ask for, the
subscription decides what it does ask for. Asking for something outside the
ACL is refused, which is the first failure mode below.

## Break it

With Play running, scroll to **Break it**. Each failure mode is one thing
going wrong on purpose. Press **Break it**, read what the card tells you to
look for, then press **Reset** before trying the next one. The diagram marks
the node each failure is about while it is in effect.

### 1. Subscribe outside the ACL

The baggage user asks for `acme/air/flight/>`. It connects, because the
client profile permits that. The subscription is refused with a 403, because
the ACL profile does not. Capability and topic authority are separate
decisions, and this is what that separation looks like when it is enforced.

The same thing from a terminal:

```bash
bash cockpit/apps/run.sh pubsub subscribe \
  --role baggage --user svc-acme-air-baggage --sub "acme/air/flight/>"
```

### 2. A subscriber goes offline

The baggage subscriber stops while the publisher keeps going. Leave it down
for a round or two, then reset. It resumes from the next message published.
The ones it missed are gone: no queue was involved, so nothing was stored.

This is why the Play sequence starts the subscribers before the publisher.

## Try this

- Run the subscriber by hand with `acme/air/baggage/>`, then with
  `acme/air/*/*/v1/AC8763`. Predict what each one receives before you run it.
  The second is refused: the ACL only covers the baggage domain.
- Check **Connected Clients** under **On the broker** while everything runs.
  For every three messages the broker receives from the publisher, it sends
  one to the subscriber.

Next: [Fan-out](30-fan-out.md).
