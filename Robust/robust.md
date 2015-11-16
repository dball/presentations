# Writing reliable and robust services

Robust: able to withstand or overcome adverse conditions

Reliable: consistently good in quality or performance; able to be trusted

## What could possibly go wrong?

1. Your process may fail at any time

2. The services on which your program relies may fail or hang

3. Your program may have bugs

## Plan for the worst

You must choose between the risk of losing work and repeating work (choose the latter)

1. If the requester cannot resend failed or unacknowledged requests, record the work request immediately

2. Never respond until you've finished your work or recorded receipt to persistent storage

3. Find or synthesize a unique identifier for each request

4. Write your database changes in a transaction. If your work request yields very large database transactions, consider doing the work in smaller batches.

5. When remote systems provide for idempotence, use it. Otherwise, consider recording the remote system calls made by your process
   to reduce the risk of repeated side effects if your request is repeated.

6. Everything may fail or hang: mysql, redis, the filesystem, the network

7. Provide the ability to reprocess requests matching some criteria

## Automate all the things

1. Devise system invariants, and write monitors that alert when they fail. Test them before you rely on them.

2. When your program has external side effects, if the remote system can provide aggregate statistics, collect them periodically and validate them against your work register.

3. Consider injecting validation requests at a slow rate into your system and monitor their results individually.

4. Whenever you have to manually intervene, consider how you could have automated your response