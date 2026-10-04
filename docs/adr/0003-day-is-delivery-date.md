# A Day is the date a Post was delivered, not published

A Post belongs to the Day on which we received it, in the User's timezone, not the Day its Instagram `created_at` falls on. Muse runs once a day and sends everything since its previous run, so grouping by publish date would file most of each run under yesterday, into a Feed the User has already finished. Grouping by delivery date makes each Day roughly one Muse run's worth of Posts, and a finished Day never grows.

## Consequences

- Posts redelivered after a failure land in the next Day.
