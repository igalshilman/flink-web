---
layout: post
title:  "Stateful Functions 2.2.0 Release Announcement"
date: 2020-09-28T08:00:00.000Z
categories: news
authors:
- igalshilman:
  name: "Igal Shilman"
  twitter: "IgalShilman"
---

The Apache Flink community is happy to announce the release of Stateful Functions (StateFun) 3.0.0!  This release focuses on bringing remote function to the front and center of StateFun. It is now easier, more efficient, and more ergonomic to write applications that live in a separate process to the StateFun/Flink cluster.

The binary distribution and source artifacts are now available on the updated [Downloads](https://flink.apache.org/downloads.html)
page of the Flink website, and the most recent Python SDK distribution is available on [PyPI](https://pypi.org/project/apache-flink-statefun/).
For more details, check the complete [release changelog](https://issues.apache.org/jira/secure/ReleaseNote.jspa?version=12348822&styleName=&projectId=12315522) 
and the [updated documentation](https://ci.apache.org/projects/flink/flink-statefun-docs-release-3.0/).
We encourage you to download the release and share your feedback with the community through the [Flink mailing lists](https://flink.apache.org/community.html#mailing-lists)
or [JIRA](https://issues.apache.org/jira/browse/)!
{% toc %}

## Background

Starting with the first StateFun release, our focus was: **making scalable stateful applications, easy to build and run**. 
  
The first StateFun version introduced an SDK that allowed writing stateful functions that are packaged and deployed as a very specific Flink application (a StateFun application), this application is submitted to a Flink cluster. Having functions executing within the same JVM as Flink, has some advantages such as performance, and an immutable deployment, however it had few limitations:

1. ❌ Functions can be written only in a JVM based language.
2. ❌ A blocking call/CPU heavy task in one function, can affect other functions, 	and operations that need to complete in a timely manner like checkpointing.
3. ❌ Deploying a new version of the function, required a stateful upgrade of the StateFun job.

With StateFun 2.0, the community introduced the concept of “remote functions”, together with an additional SDK for the Python language.
A remote function is a function that executes in a separate process and is invoked via HTTP by Apache Flink.
Remote functions Introduced a new and exciting capability: **state and compute disaggregation** - allowing to scale the functions independently from the Flink cluster that drives messaging and state.

While addressing limitation (1) and (2) mentioned above, we still had room to improve:
  
1. ✅ Functions can be written in any language (initially Python)
2. ✅ Blocking calls/ CPU heavy computations happens in a separate process(es) 
3. 🟡 Registering a new function, or changing the state definitions of an existing function required a stateful upgrade of the StateFun Flink job.
4. ❌ The SDK had few friction points around state and messaging ergonomics - It had mostly relied on Google’s Protocol Buffers for it’s multi-language object representation.
 

With **StateFun 3.0** release, the community has enhanced the remote functions protocol (the protocol that describes how StateFun communicates with the remote process) to address the issues above.
Building on the new protocol we rewrote the Python SDK, and introduced a brand new remote Java SDK.

## New SDK
One of the goals that we set up to achieve with the new SDK is a unified set of concepts across all the languages.
Here is the same function written in Python and Java:

Python

```python
@functions.bind(typename="example/greeter", specs=[ValueSpec(name="visits", type=IntType)])
async def greeter(context: Context, message: Message):
    # update the visit count.
    visits = context.storage.visits or 0
    visits += 1
    context.storage.visits = visits

    # compute a greeting
    name = message.as_string()
    greeting = f"Hello there {name} at the {visits}th time!"

    caller = context.caller

    context.send(message_builder(target_typename=caller.typename,
                                 target_id=caller.id,
                                 str_value=greeting))
```

Java

```java
   static final class Greeter implements StatefulFunction {

        static final ValueSpec<Integer> VISITS = ValueSpec.named("visits").withIntType();

        @Override
        public CompletableFuture<Void> apply(Context context, Message message){
            // update the visits count
            int visits = context.storage().get(VISITS).orElse(0);
            visits++;
            context.storage().set(VISITS, visits);

            // compute a greeting
            var name = message.asUtf8String();
            var greeting = String.format("Hello there %s at the %d-th time!\n", name, visits);

            // reply to the caller with a greeting
            var caller = context.caller().get();
            context.send(MessageBuilder.forAddress(caller)
                    .withValue(greeting)
                    .build()
            );

            return context.done();
        }
    }
```

Although there are some language specific differences, the terms and concepts are the same:

* an address scoped storage acting as a key-value store for a particular address.
* A unified cross-language way to send/receive/ and store values across languages.
* `ValueSpec` to describe the state name, type and possibly expiration. Please note that it is no longer necessary to declare the state a head of time in a `module.yaml`.

    
For a detailed SDK tutorial, we would like to encourage you to visit:

- [Java SDK showcase](https://github.com/apache/flink-statefun-playground/tree/dev/java/showcase)
- [Python SDK showcase](https://github.com/apache/flink-statefun-playground/tree/dev/python/showcase)

## Dynamic Registration of State and Functions

Starting with this release, it is now possible to dynamically register new functions, without going trough a stateful upgrade cycle in Apache Flink.
Starting with the StateFun 3.0, it is possible to specify template 


```yaml
   endpoints:
          - endpoint:
              meta:
                kind: http
              spec:
                functions: example/*
                urlPathTemplate: https://loadbalancer.svc.cluster.local/{function.name}
            
```


## Summary

This release


1. ✅ Functions can be written in any language 
2. ✅ Blocking calls/ CPU heavy computations happens in a separate process(es) 
3. ✅ Registering a new function, or changing the state definitions of an existing function happens dynamically.
4. ✅ There is a unified, cross language type system, along with few builtin primitive types, that can be used for messaging and state.


## Important Patch Notes

Below is a list of user-facing interface and configuration changes, dependency version upgrades, or removal of supported versions that would be
important to be aware of when upgrading your StateFun applications to this version:

* [[FLINK-18812](https://issues.apache.org/jira/browse/FLINK-18812)] The Flink version in StateFun 2.2 has been upgraded to 1.11.1.
* [[FLINK-19203](https://issues.apache.org/jira/browse/FLINK-19203)] Upgraded Scala version to 2.12, and dropped support for 2.11.
* [[FLINK-19190](https://issues.apache.org/jira/browse/FLINK-19190)] All existing metric names have been renamed to be camel-cased instead of snake-cased, to conform with the Flink metric naming conventions. **This breaks existing deployments if you depended on previous metrics**.
* [[FLINK-19192](https://issues.apache.org/jira/browse/FLINK-19192)] The connection pool size for remote function HTTP requests have been increased to 1024, with a stale TTL of 1 minute.
* [[FLINK-19191](https://issues.apache.org/jira/browse/FLINK-19191)] The default max number of asynchronous operations per JVM (StateFun worker) has been decreased to 1024.

## Release Notes

Please review the [release notes](https://issues.apache.org/jira/secure/ReleaseNote.jspa?projectId=12315522&version=12348350)
for a detailed list of changes and new features if you plan to upgrade your setup to Stateful Functions 2.2.0.

## List of Contributors

The Apache Flink community would like to thank all contributors that have made this release possible:

TOOD: complete

If you’d like to get involved, we’re always [looking for new contributors](https://github.com/apache/flink-statefun#contributing).


