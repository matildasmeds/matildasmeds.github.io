+++
title = 'My logging recipe for server side Rust'
date = 2024-04-11T21:15:09+02:00
draft = false
+++

In this post I show one way of instrumenting your backend code, using the [tracing crate](https://docs.rs/tracing/latest/tracing/) from the [tokio ecosystem](https://tokio.rs/). We also discuss logging and what makes it useful. This post would be most useful to the reader who might already be logging, but is not yet using [tracing::instrument](https://docs.rs/tracing/latest/tracing/attr.instrument.html) attribute macro.

## What is logging?

Logs are messages emitted by your program that are used for debugging and alerting purposes. In a sense the program tells its story in the log lines emitted. Whether there are errors, warnings, general information, debugging information, all this can be expressed as logs. Logging can be considered a key pillar in any Observability stack, alongside with metrics and traces.

Here's a succinct [definition](https://www.baeldung.com/cs/trace-vs-log) by Amanda Viescinski:

> A log is a time-stamped record of discrete events that have occurred over time in a system. Therefore, a log provides an ordered history of the occurrence of events in a system.

### Easy retrieval is necessary for logs to be useful

To be useful in real life, you need to be able to find and group relevant log lines easily. In practice, this means using relevant identifiers and spans. Without easy retrieval logging is much less useful, as anyone who has been there can testify. We will discuss identifiers and spans in a moment.

Eventually, to be able to view and search logs in production, you need to send them to a log management system. That is beyond the scope of this article. We will discuss how to make your logs useful, and how to do that in async Rust, using tokio, tracing and [tracing-subscriber](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/) crates.

### Relevant identifiers to filter log lines

Identifiers link the log line to a particular user, organization, resource.

Let's think about the typical scenarios when one needs to dig into the logs. More often than not, a particular user has encountered an issue while using the application. Ideally, we should be able to pull all the log lines for that user to start hunting for clues. That is why I think the most important identifier in logging is generally the user_id. 

When we search logs with the user_id (or similar), we find the answer to the question, what are all the things that have happened to this user. With a bit of luck we will see an error that gives enough hints to resolve the issue.

### Unit of work

In backend development, a natural unit of work is a request made to our own server. There could be smaller units of work that one might want to track separately, for instance a particular method that validates order items on a purchase order, or larger ones, comprising several requests, such as completing different steps when signing up for a service.

To keep it simple though I recommend starting with the request as the basic unit of work, and then adding on top of that if needed. So, in addition to the user_id, we now have request_id as well, which helps us group the log lines per request.

For sub-request units of work we can use any identifier that makes sense in that context.

For a work unit compassing several requests the happiest scenario is when one can use an identifier that occurs naturally in the system, for instance a match_id, if it is a game, or a sales_order_id for order management software.
### Use spans to represent units of work

According to tracing crate's [documentation](https://docs.rs/tracing/latest/tracing/span/index.html):
> Spans represent periods of time in which a program was executing in a particular context.

Also,
> As a rule of thumb, spans should be used to represent discrete units of work (e.g., a given request’s lifetime in a server) or periods of time spent in a given context (e.g., time spent interacting with an instance of an external system, such as a database).

Span is a named scope that represents a unit of work. It maps to a specific method in your code, and then it shows what was the context of a particular log line. Any descriptive name is good, as long as it identifies conceptually what the span is about, and is distinct from all other spans.

When a span has fields, these are added to each log line emitted in that span. These fields can carry identifiers, but they can also record outcomes and durations and other data that is determined during the execution of the span. Log lines can be searched and grouped easily, using the data included in the fields.

Spans can also be nested if needed. For instance, in the order management context, we could have "SalesOrder" as the parent span, and "place_order" and "confirm_order" as child-spans, or "SalesOrder::place_order" and "SalesOrder::confirm_order" as two separate, non-nested spans.

> In a nutshell: Span represents a unit of work.
> Identifiers link the span (with its log lines) to resources: user, organization, match, purchase order, and so on.

## Let's put it to practice

You will need both the [tracing crate](https://docs.rs/tracing/latest/tracing/), and [tracing-subscriber crate](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/).
The first to emit, and the second to process tracing events.

### Set up tracing-subscriber

To actually see the emitted log lines, you need to set up the tracing-subscriber. Place this initialization code somewhere in the initialization phase of your program.

```rust
tracing_subscriber::fmt()
  .init();
```

This is the quickest way to initialize the tracing-subscriber.

There are plenty of useful opt-in features though. For instance, you can format your lines as json, and include the moments when a span is created and exited, like so:

 ```rust
tracing_subscriber::fmt()
  .json()
  .with_span_events(FmtSpan::NEW | FmtSpan::EXIT)
  .init();
```

For this example to work, remember to add the required opt-in features in your Cargo.toml as well.

```toml
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

### Instrument async methods with tracing crate

Creating a span for an async method is easy, using the attribute macro [instrument](https://docs.rs/tracing/latest/tracing/attr.instrument.html)

Let's assume an implementation for a game of TicTacToe, and a method called process_turn. We can add player_id and match_id as fields on the Span, like so:

```rust
struct TicTacToe {}

impl TicTacToe {
  #[tracing::instrument(name = "TicTacToe::process_turn", skip_all, fields(match_id = %match_id, player_id = %player_id))]
  pub async fn process_turn(&self, match_id: &MatchId, player_id: &PlayerId, turn: &PlayerTurn) -> Result<UpdatedBoard, Error> {
    // validate parameters, update match, etc
    // emit relevant log lines, for instance
    tracing::info!("Turn processed successfully!")
    // return updated board
  }
}
 ```

With this, a span is opened, with fields for match_id and player_id. All log lines emitted between entering this method and exiting it, including in the calls made within the method, will now be part of the same span, with these fields recorded for each line. It is enough to fish for log lines related to a particular game or user, to gain qualitative insight to what has happened in that game.

With **name** we set the name for the span. By default this kind of span records all the parameters of the function. With **skip_all** you can skip them all, as usually this is not needed. You can also skip any parameters individually with **skip**. I like **skip_all** however, because that way I can amend the method signature without touching the attribute macro.

**fields** declares what fields can be recorded on the span. Note that when setting their values, we can evaluate statements, so that exact values can be extracted and formatted exactly as we please. You can even create a new identifier if you want, for instance like this, using [uuid crate](https://docs.rs/uuid/latest/uuid/).

```rust
#[tracing::instrument(name = "process_request", fields(request_id = Uuid::new_v4().to_string())]
async fn process_request(req: Request) -> Result<Response, Error>
{
    // process request
}
```

### How to record values after creating the span?

How about the times, when you want to include a value for a field that is not yet available when creating the span. An outcome or duration can be recorded only when the value is known. In these cases, we need to declare the field when creating the span and record the value later.

```rust
#[tracing::instrument(name = "process_request", fields(request_id = Uuid::new_v4().to_string(), duration)]
async fn process_request(req: Request) -> Result<Response, Error>
{
    let start = std::time::Instant::now();
    // process request
    tracing::Span::current().record("duration", start.elapsed());
}
```

This adds a field called duration on the span. Then in the method we access the current span, and record a value on the field at the end of the method. The duration will appear on the span exit, or any log line we emit on the span after recording the value. Note that all fields need to be declared when creating the span, otherwise the field won't show on the span.

### Word of warning: How not to create spans in async Rust

I made my first attempt at logging async code in Rust using span.enter(). Sadly, without having read the ample warnings for this [method](https://docs.rs/tracing/latest/tracing/struct.Span.html#method.enter), on tracing documentation.

```rust
let _entered = span.enter();
```

There are constructs in Rust, that are not safe in async code, and the span guard [Entered](https://docs.rs/tracing/latest/tracing/span/struct.Entered.html) is one of those. It may look innocent, but it is hard to handle correctly.

Of course I held the span guard across awaits, and then marvelled at why things looked good on my machine, but not on multi-user environments. Some log lines would not have spans at all, while others would have a cascade of spans not belonging to that line. If the spans don't end up on the lines where they belong, they are useless.

The bigger lesson of course is, to take a good look at the documentation of any new methods you want to use in async code.

Having burnt my fingers with the span guard, I was so happy to discover the recipe I present in this post. It is easy and works great for async Rust.
### Final words

I find these two crates flexible and easy to use. There's more to discover in the docs, and plenty of opportunities for fine tuning your logs.

For learning more about observability in Rust, I want to give a whole-hearted recommendation for Luca Palmieri's [rust-telemetry-workshop](https://github.com/mainmatter/rust-telemetry-workshop). This open-sourced repository is a test driven learning resource, that has been really useful for me to get hit the ground running with observability, using Rust. Each small chapter explores one or two concepts at a time. You will need to make the tests pass to move on to the next exercise. A bit like good ole rustlings, but for observability!

All in all, I hope this post has been useful for you! Let me know what you think, if there is anything that was not clear or something that you would like to know more about. Until next time!
