# Machine logs

> The node-local logs of the start commands of machines: their retention, files, writes, and reads.

The log of a machine holds the output of its start command, line by line. The guest agent reads each line and sends it to the node over the guest link ([Guest link](/reference/guest-link/index.html.md)). The node keeps the log in local files. `StreamMachineLogs` of the machine API reads it ([the machine API](/reference/machine-api/index.html.md#streammachinelogs)). The code is in `internal/node/machinelog`.

## Retention

| Item | Value |
| --- | --- |
| Size of the log of one machine | at most 100 MiB (`machinelog.DefaultMaxBytes`), plus the segment that takes the writes |
| Age | 7 days (`machinelog.DefaultMaxAge`), from the last line of a segment |
| Size of a segment | 8 MiB (`machinelog.DefaultSegmentBytes`). The next line starts a new segment. |
| Delete of the machine | The executor drops the whole log once the control role accepted the delete. A line of the machine can still wait in the queue. For 1 hour, the store drops such lines, so the log does not come back. |

A segment goes once the log holds more than 100 MiB, oldest first. The segment that takes the writes stays. A segment whose last line is older than 7 days goes too. The node checks the age after each line, and every hour for every machine. A machine without a segment loses its directory.

## Files

The node keeps each log in `<data_dir>/logs/<machine-id>/`. Each segment is a file named by its number, with 20 digits, such as `00000000000000000001.log`. The numbers grow from 1.

A segment holds records. Each record is a varint length and then one `hopbox.guest.v1.LogEntry`: the time, the stream (stdout or stderr), the line without its newline, and the count of lines that were lost before it. A record has at most 64 KiB. A reader stops at a record that is larger, or cut short. It logs `a machine log has a damaged record` at level warn, with `machine`, `segment`, and `err`.

After a restart of `hopboxd`, the node never adds to an old segment. A crash can cut the last record of a segment short, so the first new line starts a new segment. A write can fail, for example on a full disk. Then the store cuts the segment back to its last whole record. The next line goes into the same segment.

## Writes

The guest link hands each line to the store. The store never waits for the disk there, also not while the writer waits for a slow disk. A queue holds up to 4096 lines for all machines. When the queue is full, the store drops the line. The next line that it writes for the machine counts the lost line in its field `dropped`. A write that fails counts the line as dropped too. It logs `cannot write the log of a machine` at level warn, at most once a minute, with `machine`, `err`, and `failures_since_last_warning`.

## Reads

A read takes these options:

| Option | Description |
| --- | --- |
| `Tail` | Sends only the last lines, at most 10000 (`machinelog.MaxTail`). 0 sends all. |
| `Since` | Sends only the lines from this time on. |
| `Follow` | Sends each new line after the old ones, until the client ends the read. A read that follows also follows into new segments. The delete of the machine ends it. |

A segment that a read did not reach yet, and that the retention drops, reads as empty.

A read holds one line at a time in memory. For a tail, a first pass keeps only the places of the last lines in the files, 16 bytes each. A second pass sends from the first of them. So a tail of 10000 lines holds 160 KB, whatever the size of the lines. The first pass reads the whole log, up to 100 MiB.

## Logs

The store writes these records with `component=machinelog`:

| Message | Level | Fields |
| --- | --- | --- |
| `cannot write the log of a machine` | warn | `machine`, `err`, `failures_since_last_warning`. At most once a minute. |
| `cannot drop an old segment of a machine log` | warn | `dir`, `err` |
| `cannot read the machine logs` | warn | `dir`, `err` |
| `cannot read a machine log` | warn | `machine`, `err` |
| `a machine log has a damaged record` | warn | `machine`, `segment`, `err` |
