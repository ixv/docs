---
navhome: /docs/
navuptwo: true
next: true
sort: 1
title: Hall architecture
---

# Hall architecture

This document is complemented by Hall's source code, but doesn't require it. Definitions of data structures, code snippets and diagrams will be provided where useful.

Hall's implementation is structured according to the new Gall model. Familiarity with its concepts is assumed. See [the new Gall model](../new-gall).

## Table of contents

* [Introduction](#introduction)
* [Structures & functionality](#structures--functionality)
  * [Circles](#circles)
  * [Messages](#messages)
  * [Participant metadata](#participant-metadata)
  * [Configurations](#configurations)
  * [Stories](#stories)
  * [General use](#general-use)
  * [Federation](#federation)
  * [Public membership](#public-membership)
* [Interfaces for applications](#interfaces-for-applications)
  * [Interactions](#interactions)
  * [Queries, prizes and rumors](#queries-prizes-and-rumors)
* [Communication between Halls](#communication-between-halls)
  * [Interactions](#interactions-1)
  * [Querying](#querying)
  * [Federation](#federation)
* [Hall implementation](#hall-implementation)
  * [Subscriptions](#subscriptions)
  * [Rumors](#rumors)
  * [Messaging](#messaging)
* [The future](#the-future)
  * [Features and functionality](#features-and-functionality)
* [Further reading](#further-reading)


## Introduction

Talk was Urbit's first big user-facing application. It continues to enjoy a prominent role in the Urbit landscape, but does so as two separate applications this time.

The messaging parts of Talk have been separated from its user interface parts. What we ended up with is a shiny new generic messaging bus, and the chat interface we all know and love. The messaging bus, which should now prove useful to many different applications, will be named `Hall`. Applications that use it are referred to as *clients*. One such application, as you might have guessed, is `Talk`.


## Structures & functionality

### Circles

```
++  circle     {hos/ship nom/term}                      :<  native target
```

A `circle` is essentially a named collection of messages created by and hosted on a ship's Hall, usually represented as `~ship-name/circle-name`. Most of Hall revolves around doing things with circles.

### Messages

When we subscribe to a circle, the primary thing we're interested in is its messages. Message data itself isn't that complicated, but a fair amount of metadata comes into play when actually sending a message. Let's work our way up, starting at the contents.

```
++  speech                                              :>  content body
  $%  {$lin pat/? msg/cord}                             :<  no/@ text line
      {$url url/purf:eyre}                              :<  parsed url
      {$exp exp/cord res/(list tank)}                   :<  hoon line
      {$ire top/serial sep/speech}                      :<  in reply to
      {$fat tac/attache sep/speech}                     :<  attachment
      {$app app/term sep/speech}                        :<  app message
      {$inv inv/? cir/circle}                           :<  inv/ban for circle
  ==                                                    ::
```

At the heart of every message lies a `speech` that describes the message body. There's a large number of different speech types, from simple text messages to parsed URLs, Hoon expressions and more.

```
++  audience   (set circle)                             :<  destinations
++  serial     @uvH                                     :<  unique identifier
++  thought                                             :>  inner message
  $:  uid/serial                                        :<  unique identifier
      aud/audience                                      :<  destinations
      wen/@da                                           :<  timestamp
      sep/speech                                        :<  content
  ==                                                    ::
```

A `speech` is always accompanied by various bits of metadata. We include a unique identifier, a message timestamp and an `audience`, the set of all intended recipients of the message.

```
++  telegram   {aut/ship thought}                       :<  whose message
++  envelope   {num/@ud gam/telegram}                   :<  outward message
```

Finally, before sending the message over the wire, we add on the message's original author. In most cases, we also include a sequence number, the index of the message in the originating circle's list of messages. This can be a useful point of reference for clients.

### Participant metadata

Messages aren't the only thing a subscription gets us. We're also kept up to date with relevant metadata.

```
++  crowd      {loc/group rem/(map partner group)}      :<  our & srcs presences
++  group      (map ship status)                        :<  presence map
++  status     {pec/presence man/human}                 :<  participant
++  presence                                            :>  status type
  $?  $gone                                             :<  absent
      $idle                                             :<  idle
      $hear                                             :<  present
      $talk                                             :<  typing
  ==                                                    ::
++  human                                               :>  human identifier
  $:  han/(unit cord)                                   :<  handle
      tru/(unit truename)                               :<  true name
  ==                                                    ::
++  truename   {fir/cord mid/(unit cord) las/cord}      :<  real-life name
```

`status` is user-set metadata that describes, well, the status of users in a circle. This encompasses their `presence`, which shows their activity, and their `human` identity, which includes their display handle.  
For reasons we'll discover shortly, circles keep track of both their own `group` and those of the circles they're subscribed to. `crowd` encapsulates this.

### Configurations

```
++  lobby      {loc/config rem/(map circle config)}     :<  our & srcs configs
++  config                                              :>  circle config
  $:  src/(set circle)                                  :<  active sources
      cap/cord                                          :<  description
      fit/filter                                        :<  message rules
      con/control                                       :<  restrictions
  ==                                                    ::
++  filter                                              :>  content filters
  $:  cas/?                                             :<  dis/allow capitals
      utf/?                                             :<  dis/allow non-ascii
  ==                                                    ::
++  control    {sec/security ses/(set ship)}            :<  access control
```

Another part of metadata we get from circle subscriptions is configurations. Again, circles want to remember their own configuration, as well as those of their subscriptions. But why, precisely?

The `config` structure contains `src`, a set of partners. These indicate the different sources a circle is currently pulling content from. This allows circles to aggregate messages from multiple places. In doing so, it also receives metadata from those places, hence why we have structures for storing "remote" presences and configurations alongside local ones.

Aside from that, `config` contains a description for the circle, which is exactly what it sounds like. It also specifies a `filter`, which are content formatting rules all messages passing through this circle are made to adhere to.

Lastly, there's a `control` structure that contains both a security mode and a list of ships, which is either a white- or blacklist depending on the aforementioned mode. There are four such modes available.

```
++  security                                            :>  security mode
  $?  $channel                                          :<  blacklist
      $village                                          :<  whitelist
      $journal                                          :<  pub r, whitelist w
      $mailbox                                          :<  our r, blacklist w
  ==                                                    ::
```

A `channel` is publicly readable and writable, with a blacklist for blocking.  
A `village` is privately readable and writable, with a whitelist for inviting.  
A `journal` is publicly readable and privately writable, with a whitelist for authors.  
A `mailbox` is readable by its owner and publicly writable, with a blacklist for blocking.

### Stories

To see how that all ties together, we're going to take a look at Hall's state.

```
++  state                                               :>  hall state
  $:  stories/(map term story)                          :<  conversations
      ::  ...                                           ::
  ==                                                    ::
++  story                                               :>  wire content
  $:  grams/(list telegram)                             :<  all messages
      locals/group                                      :<  local presence
      remotes/(map circle group)                        :<  remote presence
      shape/config                                      :<  configuration
      mirrors/(map circle config)                       :<  remote config
      ::  ...                                           ::
  ==                                                    ::
```

Stories are the primary driver behind Hall. They are the structures that are used to power circles, which we can now say are named stories hosted on ships.

With the configuration described above in mind, we can try and imagine the things we can do with stories. Knowing that we can subscribe them to any number of sources, they can function as central hubs for our messaging, aggregate specific kinds of data feeds, or simply accept whatever messages get sent to it like a regular old chatroom.

### General use

Upon initial startup, Hall creates a default story, a mailbox named `inbox`. This mailbox is the primary target for anyone and anything that wants to reach its owner. Applications can use it to send notifications and other information to the user (Hall itself does this as well), and users can use it to send direct messages to each other.

Applications that use Hall, especially clients, are encouraged to use the default mailbox as the primary messaging hub. This way, users can easily switch between different applications without "losing" their subscriptions, message backlog, etc.

As an example, Talk operates like this, serving as an interface for reading from and managing the user's mailbox. Local stories are subscribed to through the `inbox`, which ends up containing all messages the user receives.

### Federation

Some places, like `urbit-meta`, will want to be easily accessible to everyone on the network, and actually support the load that comes with that. Both these desired can be met through federation.

In federation, stories and any changes to them trickle down from ~zod, to other galaxies, to their stars. Only changes to messages and presence can also move upstream, from stars, to galaxies, to ~zod (and then back down again).  
This way, all planets and comets will be able to subscribe to `/urbit-meta` (that is, the `urbit-meta` circle on their parent ship) and get all updates on the story, regardless of where on the network they originated.

### Public membership

To aid in discoverability of circles, it is possible for users to make their participation in any given circle public. This data can then be used by things like profile widgets, or read out directly by other users.


## Interfaces for applications

Applications can interact with Hall in two complementary ways: they can tell it what actions to perform, and they can subscribe to its state and the changes made to it.

### Interactions

Hall can be commanded by poking it with `action`s.

```
++  action                                              :>  user action
  $%  ::  circle configuration                          ::
      {$create nom/term des/cord sec/security}          :<  create circle
      {$delete nom/term why/(unit cord)}                :<  delete + announce
      {$depict nom/term des/cord}                       :<  change description
      {$filter nom/term fit/filter}                     :<  change message rules
      {$permit nom/term inv/? sis/(set ship)}           :<  invite/banish
      {$source nom/term sub/? srs/(set source)}         :<  un/sub to/from src
      ::  messaging                                     ::
      {$convey tos/(list thought)}                      :<  post exact
      {$phrase aud/audience ses/(list speech)}          :<  post easy
      ::  personal metadata                             ::
      {$notify aud/audience pes/(unit presence)}        :<  our presence update
      {$naming aud/audience man/human}                  :<  our name update
      ::  changing shared ui                            ::
      {$glyph gyf/char aud/audience bin/?}              :<  un/bind a glyph
      {$nick who/ship nic/cord}                         :<  new identity
      ::  misc changes                                  ::
      {$public add/? cir/circle}                        :<  show/hide membership
  ==                                                    ::
```

The largest part of these actions concern themselves with managing local circles. `nom` is always the name used to identify the relevant story. To disambiguate between "add" and "delete" type actions (for permissions and subscriptions), a loob `?` is used.

To send messages, two interfaces are available. `%convey` lets you specify all details of the messages, including its timestamp, serial, and fully assembled audience. `%phrase`, on the other hand, takes care of that for you, allowing you to specify just the target partners and message contents.

`%notify` and `%naming` are useful for setting your own presence and nickname in a circle respectively, so others can see if you're active and what to call you by.

"Shared UI" encompasses UI data that should be consistent across applications. For example, if the user sets a local nickname for an identity, they expect to see that nickname regardless of the application they're currently using. The same goes for glyph bindings, for easy audience targeting.

### Queries, prizes and rumors

To receive data from Hall, applications will have to query it. There are two paths that are useful for clients to query. Let's look at them and their results.

```
++  query                                               :>  query paths
  $%  {$client $~}                                      :<  shared ui state
      $:  $circle                                       :>  story query
          nom/naem                                      :<  circle name
          wat/(set circle-data)                         :<  data to get
          ran/range                                     :<  query duration
      ==                                                ::
      ::  ...                                           ::
  ==                                                    ::
++  circle-data                                         :>  kinds of circle data
  $?  $grams                                            :<  messages
      $group-l                                          :<  local presence
      $group-r                                          :<  remote presences
      $config-l                                         :<  local config
      $config-r                                         :<  remote configs
  ==                                                    ::
++  range                                               :>  inclusive msg range
  %-  unit                                              :<  ~ means everything
  $:  hed/place                                         :<  start of range
      tal/(unit place)                                  :<  opt end of range
  ==                                                    ::
++  place                                               :>  range indicators
  $%  {$da @da}                                         :<  date
      {$ud @ud}                                         :<  message number
  ==                                                    ::
++  prize                                               :>  query result
  $%  $:  $client                                       :<  /client
          gys/(jug char (set partner))                  :<  glyph bindings
          nis/(map ship cord)                           :<  nicknames
      ==                                                ::
      {$circle burden}                                  :<  /circle
      ::  ...                                           ::
  ==                                                    ::
++  burden                                              :<  full story state
  $:  gaz/(list telegram)                               :<  all messages
      cos/lobby                                         :<  loc & rem configs
      pes/crowd                                         :<  loc & rem presences
  ==                                                    ::
```

To be clear, the paths for those queries are `/client` and `/circle/[name]/[data]/[start]/[end]`, where data is at least one `circle-data`, and `start` and `end` are optional and can be either a date or a message number.  
`/client` queries produce shared UI state. That is, glyph bindings and nicknames.  
`/circle` queries produce the entire public state of the requested story, including remotes.

Any changes to the results of these queries are communicated via the following `rumor`s. It's a fairly long list of potential changes,

```
++  rumor                                               :<  query result change
  $%  {$client rum/rumor-client}                        :<  /client
      {$circle rum/rumor-story}                         :<  /circle
      ::  ...                                           ::
  ==                                                    ::
++  rumor-client                                        :<  changed ui state
  $%  {$glyph diff-glyph}                               :<  un/bound glyph
      {$nick diff-nick}                                 :<  changed nickname
  ==                                                    ::
++  diff-glyph  {bin/? gyf/char aud/audience}           :<  un/bound glyph
++  diff-nick   {who/ship nic/cord}                     :<  changed nickname
++  diff-story                                          :>  story change
  $%  {$new cof/config}                                 :<  new story
      {$config cir/circle dif/diff-config}              :<  new/changed config
      {$status cir/circle who/ship dif/diff-status}     :<  new/changed status
      {$remove $~}                                      :<  removed story
      ::  ...                                           ::
  ==                                                    ::
++  rumor-story                                         ::>  story rumor
  $?  diff-story                                        ::<  both in & outward
  $%  {$gram nev/envelope}                              ::<  new/changed msgs
  ==  ==                                                ::
++  diff-config                                         :>  config change
  $%  {$full cof/config}                                :<  set w/o side-effects
      {$source add/? src/source}                        :<  add/rem sources
      {$caption cap/cord}                               :<  changed description
      {$filter fit/filter}                              :<  changed filter
      {$secure sec/security}                            :<  changed security
      {$permit add/? sis/(set ship)}                    :<  add/rem to b/w-list
      {$remove $~}                                      :<  removed config
  ==                                                    ::
++  diff-status                                         :>  status change
  $%  {$full sat/status}                                :<  fully changed status
      {$presence pec/presence}                          :<  changed presence
      {$human dif/diff-human}                           :<  changed name
      {$remove $~}                                      :<  removed status
  ==                                                    ::
++  diff-human                                          :>  name change
  $%  {$full man/human}                                 :<  fully changed name
      {$handle han/(unit cord)}                         :<  changed handle
      {$true tru/(unit truename)}                       :<  changed true name
  ==                                                    ::
```

These are, as enforced by Gall, the changes as they happen. This makes it possible for very minimal Hall clients to be implemented that only display a stream of changes, rather than keeping state themselves.


## Communication between Halls

Halls can communicate with other Halls to send commands and subscription updates.

### Interactions

To request changes to a story, Halls can send `command`s.

```
++  command                                             :>  effect on story
  $%  {$publish tos/(list thought)}                     :<  deliver
      {$present nos/(set term) dif/diff-status}         :<  status update
      ::  ...                                           ::
  ==                                                    ::
```

A `%publish` command contains messages. These are sent to foreign Halls to be published to their stories.

The `%present` command is used for sending status updates about our ship.

### Querying

Halls can query other halls. Here's the peer move we send when we open a query on a foreign circle:

```
:*  ost.bol                                             :<  bone
    %peer                                               :<  move type
    /[our-circle]/[host]/[query-path]                   :<  rumor path
    [host %hall]                                        :<  query target
    /circle/[their-circle]/[data]/[start]/[end]         :<  query path
==                                                      ::
```

Again, the rumor path is what our Hall uses to identify what query a received message originates from. Our circle is specified so that we know what story is interested in the changes we get.  
The query path is slightly more interesting. Of course it specifies the name of their story we want to subscribe our story to, but also a "start" and "end". These can (optionally) be used to specify the range of messages we want to get from our query. Once that range has passed, we stop receiving updates.

### Federation

Along the way so far, we've skipped some parts of structures. Most of these relate to federation. Let's see what we missed.

```
++  query                                               :>  query paths
  $%  {$burden who/ship}                                :<  duties to share
      {$report $~}                                      :<  duty reports
      ::  ...                                           ::
  ==                                                    ::
++  prize                                               :>  query result
  $%  {$burden sos/(map term burden)}                   :<  /burden
      ::  ...                                           ::
  ==                                                    ::
++  rumor                                               :<  query result change
  $%  {$burden nom/term rum/rumor-story}                :<  /burden
      ::  ...                                           ::
  ==                                                    ::
++  diff-story                                          ::
  $%  {$bear bur/burden}                                :<  new inherited story
      ::  ...                                           ::
  ==                                                    ::
++  burden                                              :>  full story state
  $:  gaz/(list telegram)                               :<  all messages
      cos/lobby                                         :<  loc & rem configs
      pes/crowd                                         :<  loc & rem presences
  ==                                                    ::
```

We briefly touched upon how federation works on the higher level. On the lower, this is how things go down.

1. When Hall boots on a star or galaxy, it starts querying its parent's `/burden` path. (Galaxies query ~zod.)
2. Upon receiving that query, the parent sends a `%burden` prize containing all state of its channels (fully public circles) to the child. It also subscribes to `/report` on that child.
3. When a new story gets created on the parent, its children get a `%burden` rumor with a `%bear` story diff, and they create a local copy of that channel.
4. When something about a parent's story changes, a `%burden` rumor is sent to all its children (because they're querying `/burden`). The children apply this change to their local version of the story.
5. When a child's story has a `%grams` or `%status` change, a `%burden` rumor is sent to the parent (because they're querying `/report`). The parent applies this change to their local version of the story, and #4 happens.


## Hall implementation

Now that we know what gets sent over the wire and why, let's see what happens whenever such events happen. We won't be covering everything, but enough to illustrate the general flow.

First though, it's useful to understand how Hall's code is structured. It consists of primary cores, designed to work optimally with Gall.

* `++ta` is the transaction core. For every event that happens, be it a poke, peer or diff, the transaction core gets put to work to process it. Once it's done, it produces a list of deltas that describe how our state needs to be modified.
* `++da` is the delta application core. It is used to apply `++ta`'s deltas to our application state, and produce a list of side-effects.

Both those cores also have a core within themselves dedicated solely to dealing with stories, `++so` and `++sa` respectively.

Once all work in an engine core has finished, its `-done` arm is called to produce the result of all the core's computation.

In the diagrams that follow, regular lines indicate flows that always happen. Dashed lines indicate flows that are traversed if the described condition is met.  
Below the diagrams you will find brief explanations of the involved arms. In a particularly bright future, these might be available on-hover instead.  
For brevity, utility arms that aren't a direct and important part of the flow have been omitted.

(You will find that the origins still use old Gall arms, with some glue code in between. At the time of writing, new Gall has not yet been implemented, so this will have to do.)

We will only display the delta generation parts of the flow. Delta application is usually implemented simply enough to deduce what it does from looking at the label.

### Subscriptions

Hall-to-Hall subscriptions happen when a source gets added to a story. This is done with a `%source` action, and results in a `%peer` move being sent (prompted by a `%story %follow` delta).  
Upon receiving a valid peer on a `/circle` path, the subscribing ship is added to that circle's presence map.

![subscriptions implementation flow](https://media.urbit.org/docs/hall/diagrams/flow-subscriptions.png "subscriptions implementation flow")

```
++  poke-talk-action      :<  we got poked with an action.
  ++  ta-action           :<  applies an action.
  ++  ta-config           :<  (re)configures a story.
::                        ::
++  peer                  :<  we got peered.
  ++  g-query             :<  resolves query to prize.
  ++  ta-subscribe        :<  act upon subscription.
  ++  so-attend           :<  adds a ship's status to the story.
```

### Rumors

Once a query has opened, Hall will receive updates on it, rumors. The changes describes in these rumors are applied via the following flow.

![rumor implementation flow](https://media.urbit.org/docs/hall/diagrams/flow-rumors.png "report diff implementation flow")

```
++  diff-talk-rumor       :<  we got a query update.
  ++  ta-hear             :<  applies a rumor to the story it's intended for.
  ++  so-hear             :<  applies a %circle rumor to the story.
    ++  so-config-full    :<  splits a %full diff-config into separate changes.
    ++  so-bear           :<  accept burden, assimilate into the story's state.
    ++  so-lesson         :<  sends a report with presences.
```

### Messaging

To send messages, the user sends a `%convey` or `%phrase` action, resulting in a `%publish` command being sent to the involved partners. Receiving a `%publish` command causes its messages to be added to the story through the creation of a `%story %grams` delta.

![messaging implementation flow](https://media.urbit.org/docs/hall/diagrams/flow-messaging.png "messaging implementation flow")

```
++  poke-talk-action      :<  we got poked with an action.
  ++  ta-action           :<  applies an action.
::                        ::
++  poke-talk-command     :<  we get poked with a command.
  ++  ta-apply            :<  applies a command.
  ++  ta-think            :<  consumes each message in the given list.
  ++  ta-consume          :<  conducts a message to each partner in audience.
  ++  ta-conduct          :<  records or sends a message.
    ++  ta-transmit       :<  sends a message to a partner.
    ++  ta-record         :<  stores a message in a story.
    ++  so-learn          :<  either adds or modifies a message.
```


## The future

Hall is neither complete nor perfect. There are features that need to be implemented, varying from small quality of life changes to broad-reaching functionality.

### Features and functionality

To turn Talk into a [sustainable social platform](https://urbit.org/fora/posts/~2017.4.26..18.00.25..b93c~/), it needs a number of things.

Primarily, discoverability of circles. A possible solution for realizing this is making it possible for users to add friends, which would have their Hall subscribe to a list of circles the added friends are in. The existing public membership functionality can be leveraged for this.

Eventually, content self-moderation might need to be implemented. Users would flag their content if it is potentially offensive. Others would easily be able to filter out content that might offend them.  
This functionality brings its own large sets of challenges that need to be tackled.

Additionally, federation might be put to broader use. Knowing a circle is federated could help fall back to alternative hosts in case the one Hall originally subscribed to is unavailable.  
Implementation-wise, care would need to be taken for this to not get ugly. Not to mention, when/how would availability of a host be verified?

Other changes that might have a fair impact on the functioning of Hall are un/read states for messages, and being able to use moons to allow your friends to chat with you in a local channel.

Smaller functionality that still needs to be implemented includes:
* Extended permissions management. Being able to black-/whitelist entire ship classes.
* Improved presence functionality. Actually using the presence system, and sending "typing", "idle", etc. statuses.
* Polls.

And of course there's a lot Talk, the client, can improve on as well.


## Further reading

To gain a more thorough understanding of Hall's inner workings, take a look at its source code. It comes with inline documentation.  
[On Github.](https://github.com/urbit/arvo/blob/master/app/hall.hoon)

To see an expansive example of a Hall client, take a look at the code of Talk. It, too, comes with inline documentation.  
[On Github.](https://github.com/urbit/arvo/blob/master/app/talk.hoon)
