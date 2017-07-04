---
title: Log Replayer
keywords: lambda
sidebar: lambda_sidebar
toc: false
permalink: lambda_functions_log_replayer.html
folder: lambda
---

# Log Archiver Replay

The old way of thinking. New way I think doesn't involve ever stopping the stream but triggering the consumer functions directly from a archive replayer.

* Stop processing new logs by disabling Lambda DynamoDB stream trigger
* Wait X time (2min?)
  * At this point new entries are building up in the DB and the Stream but are not being consumed by lambda. 24 hour timer starts.
* Archive all logs where PROCESSED=TRUE and ARCHIVED=FALSE
* Purge stats database
* Start processing new logs by enabling Lambda DynamoDB stream trigger
* Wait X time (5min?)
* Start processing new logs
  * At this point Lambda should start triggering and consuming the stream and marking lines as processed.
* Replay necessary logs into incomming logs with PROCESSED=FALSE and ARCHIVED=TRUE


* Write Timestamp CUTOFF_TS to a DynamoDB "Meta" table
* Wait a few minutes for any currently processing nodes to finish
* Force Archive all processed log entries
* Trigger log_replayer



## log_replayer
```
Clear Stats tables
For each S3 log entry:
	Set Tag REPLAY = PENDING
	Set Tag REPLAY_LINE = 0
	Set Tag STATE_CHANGE = [Current Time]
	asyncronously execute the log_consumer for that protocol
Set up schedule for log_replayer_watch every minute
```

## log_consumer

```
Determines if this is a DynamoDB trigger or replay trigger
If DynamoDB stream process line item and exit
If replay trigger:
	Set Tag REPLAY = PROCESSING
	Set Tag STATE_CHANGE = [Current Time]
	while >= 30sec left in execution:
		read from S3 starting at REPLAY_LINE and calculate stats
			ignore all lines after CUTOFF_TS from "Meta" table
		if done with file
			local var FINISHED = true
			break
	commit data to DynamoDB stats table
		if successful
			if FINISHED == true
				Set Tag REPLAY = FINISHED
				Set Tag STATE_CHANGE = [Current Time]
			if FINISHED == false
				Set Tag REPLAY = PENDING
				Set Tag REPLAY_LINE = last processed line
				Set Tag STATE_CHANGE = [Current Time]
				asyncronously retrigger log_consumer for same S3 file
		if not successful
			Set Tag REPLAY = PENDING
			[ REPLAY_LINE remains unchanged from initial lambda trigger value ]
			Set Tag STATE_CHANGE = [Current Time]
			asyncronously retrigger log_consumer for same S3 file
```

## log_replayer_watch

```
For each S3 log entry:
	if REPLAY = FINISHED
		Remove Tag REPLAY
		Remove Tag REPLAY_LINE
		Remove Tag STATE_CHANGE
	if REPLAY = PROCESSING || PENDING
		if STATE_CHANGE older than [Lamda Timeout + a buffer]
			# Processing appears to have failed somewhere
			Set Tag REPLAY = PENDING
			Set Tag STATE_CHANGE = [Current Time]
			asyncronously retrigger log_consumer for S3 file
```

{% include links.html %}
