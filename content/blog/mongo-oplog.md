+++
title = "On MongoDB Oplog and CDC"
date = 2026-11-14
+++

This post aims to answer the following questions:

1. How exactly does MongoDB propogate the writes in the primary to the secondaries?
2. If customers want to build a CDC (Change Data Capture) pipeline out of the writes in order to perform async actions, how does MongoDB faciliate that?

I was actually researching (2) and realised the answer to this is closely tied to the answer to (1).

# Oplog

Let's start off from the absolute basics. In order to have fault tolernace and also scale your reads (in case you can't easily vertically scale your DB), you start adding secondary DBs which are supposed to hold the same data as your primary DB, but with a bit of lag. All your writes go to the primary DB, but you can configure your reads to go to the secondary DBs. Now the question is - how exactly do you propogate your changes in the primary DB to your secondary DBs? The answer to this question, which I have given away in the title, is oplog. 

From [the docs](https://www.mongodb.com/docs/manual/core/replica-set-oplog/):

> The oplog (operations log) is a special capped collection that keeps a rolling record of all operations that modify the data stored in your databases. If write operations do not modify any data or fail, they do not create oplog entries.

This seems rather simple - just store a list of all operations that have happened on the primary and the secondaries can just tail the oplog to update its own data. But let's go deeper. 

## Is an oplog stored per collection, db or something else?

It is one per member of the replica set. If you don't know what a replica set is:

1. MongoDB documents live inside a collection.
2. Collections live inside a DB.
3. DBs live inside a "replica set" which is just the fancy term for your primary + secondary machines. 
4. Your replica sets (only one if not sharded) live inside a cluster.

The important point to note here is that each member of your replica set is one machine. So all of your "primary" data in all your collections and all your DBs is stored in one machine. The oplog is also stored in this machine. This data is then replicated in other machines (called the secondaries).


## The docs say oplog is a capped collection. What does it mean?

It means the oplog is meant to of a fixed size. You can configure a maximum size, and optionally a minimum retention time. The oplog will grow past the size limit to keep that many hours. There's also another exception where the oplog can grow larger which I'll cover below.


# Change Data Capture

Let's you want to perform action async based on writes in your db. You can always update your application code to publish a message in a kafka topic. But that leads to all sorts of consistency problems and also adds latency in your sync path which you want to avoid. What if there's way to get this functionality out of MongoDB itself?

The answer is once again oplog. Apart from MongoDB using the oplog to propogate changes to secondaries, it also serves this use case. You can either directly query the oplog collection itself (which is not recommended) or you can use the change stream APIs exposed by MongoDB itself documented [here](https://www.mongodb.com/docs/manual/changestreams/?language-no-dependencies=shell). I will not be going into further details into the specifics in this post. 

But I do want to talk about one more topic which I found interesting - majority commit point.

## Majority Commit Point

Does the change stream APIs return all the data that's in the oplog of your primary? The answer is no. It only returns till something called as the majority commit point. So what this means is if a certain operation has not been copied by majority of the replica set's members, the change stream APIs will not return this operation though it's part of the primary oplog.

This is the exception I was referring to above. MongoDB does allow the oplog collection to go above the configured size limits if majority commit point is going to be rolled over.

I also find this interesting because there's a clear correlation between secondary lag and your CDC lag, which might happen in production. 

**Why does it exist?** 

It's got to do with what happens when the primary fails. When the primary fails, the secondaries have an election and whoever gets the majority votes **first** becomes the new primary. 

The relevant voting rule here is that a member refuses to vote for a candidate whose oplog is behind its own. Let's say instead of majority commit point, we were willing to advance the commit point if only one secondary had syned the operation. If the primary fails and if another secondary wins the election (possible as it could votes from secondaries whose oplog is on par or behind), we now have a primary that doesn't have what was exposed in the change stream APIs which is a disaster. Having a majority solves this problem and that's the reason that the change stream APIs only expose operations till this point. 
