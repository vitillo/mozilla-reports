---
title: new-profile ping validation on Nightly
authors:
- dexter
tags:
- tutorial
- examples
- telemetry
- spark
created_at: 2017-06-07 00:00:00
updated_at: 2017-06-12 16:01:37.633462
tldr: This notebook verifies that the 'new-profile' ping behaves as expected on the
  Nightly channel.
thumbnail: images/output_31_1.png
---
# Validate 'new-profile' submissions on Nightly
This analysis validates the 'new-profile' pings submitted by Nightly builds for one week since the 'new-profile' ping was enabled [in bug 1364068](https://bugzilla.mozilla.org/show_bug.cgi?id=1364068). We are going to verify that:

- the install ping is received within a reasonable time after the profile creation;
- we receive one ping per client;
- we don't receive many duplicates overall.


```python
import ujson as json
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np
import plotly.plotly as py
import IPython

from plotly.graph_objs import *
from moztelemetry import get_pings_properties, get_one_ping_per_client
from moztelemetry.dataset import Dataset
from datetime import datetime, timedelta

%matplotlib inline
IPython.core.pylabtools.figsize(16, 7)
```
    Unable to parse whitelist: /mnt/anaconda2/lib/python2.7/site-packages/moztelemetry/histogram-whitelists.json.
    Assuming all histograms are acceptable.


We'll be looking at the pings that have been coming in since the 31st May 2017 to the 7th June 2017.  


```python
# Note that the 'new-profile' ping needs to use underscores in the Dataset API due to bug.
pings = Dataset.from_source("telemetry") \
    .where(docType='new_profile') \
    .where(appUpdateChannel="nightly") \
    .where(submissionDate=lambda x: "20170531" <= x < "20170607") \
    .where(appBuildId=lambda x: "20170531" <= x < "20170607") \
    .records(sc, sample=1.0)
```
    fetching 5.53331MB in 1438 files...



```python
ping_count = pings.count()
ping_count
```




    3840



### How many pings were sent in-session and how many at shutdown?
The `new-profile` ping can be sent either during the browsing session, 30 minutes after the browser starts, or at shutdown ([docs](https://gecko.readthedocs.io/en/latest/toolkit/components/telemetry/telemetry/data/new-profile-ping.html#payload-reason)). Let's see how many pings we get in each case.


```python
raw_subset = get_pings_properties(pings, ["id",
                                          "meta/creationTimestamp",
                                          "meta/Date",
                                          "meta/Timestamp",
                                          "meta/X-PingSender-Version",
                                          "clientId",
                                          "environment/profile/creationDate",
                                          "payload/reason"])
```
Discard and count any ping that's missing creationTimestamp or Timestamp.


```python
def pct(a, b):
    return 100.0 * a / b

subset = raw_subset.filter(lambda p: p["meta/creationTimestamp"] is not None and p["meta/Timestamp"] is not None)
print("'new-profile' pings with missing timestamps:\t{:.2f}%".format(pct(ping_count - subset.count(), ping_count)))
```
    'new-profile' pings with missing timestamps:	0.00%



```python
reason_counts = subset.map(lambda p: p.get("payload/reason")).countByValue()

for reason, count in reason_counts.iteritems():
    print("'new-profile' pings with reason '{}':\t{:.2f}%".format(reason, pct(count, ping_count)))
```
    'new-profile' pings with reason 'startup':	24.32%
    'new-profile' pings with reason 'shutdown':	75.68%


This means that, among all the `new-profile` pings, the majority was sent at shutdown. This could mean different things:

- the browsing session lasted less than 30 minutes;
- we're receiving duplicate pings at shutdown.

### Let's check how many duplicates we've seen


```python
def dedupe(pings, duping_key):
    return pings\
            .map(lambda p: (p[duping_key], p))\
            .reduceByKey(lambda a, b: a if a["meta/Timestamp"] < b["meta/Timestamp"] else b)\
            .map(lambda pair: pair[1])

deduped_docid = dedupe(subset, "id")
deduped_docid_count = deduped_docid.count()
total_duplicates = ping_count - deduped_docid_count
print("Duplicate pings percentage (by document id): {:.2f}%".format(pct(total_duplicates, ping_count)))
```
    Duplicate pings percentage (by document id): 0.21%


The 0.21% of ping duplicates is nice, compared to ~1% we usually get from the `main` and `crash` pings. However, nowdays we're running de-duplication by document id at the pipeline ingestion, so this might be a bit higher. To check that, we have a `telemetry_duplicates_parquet` table and [this handy query](https://sql.telemetry.mozilla.org/queries/4809) that says 0 duplicates were filtered on the pipeline. This means that our 0.21% is the real duplicate rate for the `new-profile` ping on Nightly.

Did we send different pings for the same client id? We shouldn't, as we send at most one 'new-profile' ping per client.


```python
deduped_clientid = dedupe(deduped_docid, "clientId")
total_duplicates_clientid = deduped_docid_count - deduped_clientid.count()
print("Duplicate pings percentage (by client id): {:.2f}%".format(pct(total_duplicates_clientid, deduped_docid_count)))
```
    Duplicate pings percentage (by client id): 0.89%


That's disappointing: it looks like we're receiving multiple `new-profile` pings for some clients. Let's dig into this by analysing the set of pings deduped by document id. To have a clearer picture of the problem, let's make sure to aggregate the duplicates ordered by the time they were created on the client.


```python
# Builds an RDD with (<client id>, [<ordered reason>, <ordered reason>, ...])
clients_with_dupes = deduped_docid.map(lambda p: (p["clientId"], [(p["payload/reason"], p["meta/creationTimestamp"])]))\
                                  .reduceByKey(lambda a, b: sorted(a + b, key=lambda k: k[1]))\
                                  .filter(lambda p: len(p[1]) > 1)\
                                  .map(lambda p: (p[0], [r[0] for r in p[1]]))

# Check how often each case occurs.
sorted(clients_with_dupes.map(lambda p: tuple(p[1])).countByValue().items(), key=lambda k: k[1], reverse=True)
```




    [((u'shutdown', u'shutdown'), 7),
     ((u'shutdown', u'startup'), 4),
     ((u'startup', u'startup'), 3),
     ((u'startup', u'shutdown'), 3),
     ((u'shutdown', u'shutdown', u'shutdown'), 2),
     ((u'shutdown', u'shutdown', u'startup'), 1),
     ((u'shutdown', u'startup', u'shutdown'), 1),
     ((u'shutdown',
       u'shutdown',
       u'shutdown',
       u'shutdown',
       u'shutdown',
       u'shutdown'),
      1),
     ((u'shutdown', u'shutdown', u'shutdown', u'startup', u'shutdown'), 1)]




```python
collected_clientid_reasons = clients_with_dupes.collect()
```

```python
num_shutdown_dupes = sum([len(filter(lambda e: e == 'shutdown', t[1])) for t in collected_clientid_reasons])
print("Duplicate 'shutdown' pings percentage (by client id): {:.2f}%"\
      .format(pct(num_shutdown_dupes, ping_count)))
```
    Duplicate 'shutdown' pings percentage (by client id): 1.07%


The multiple pings we're receiving for the same client id are mostly `new-profile` pings with reason `shutdown`. This is not too surprising, as most of the `new-profile` pings are being sent at shutdown (75%).

But does the pingsender have anything to do with this? Let's attack the problem like this:

- get a list of the "misbehaving" clients;
- take a peek at their pings (redact client ids/ids);
- figure out the next steps.


```python
count_reason_pingsender = subset\
                            .filter(lambda p: p.get("clientId") in misbehaving_clients)\
                            .map(lambda p: (p.get("payload/reason"), p.get("meta/X-PingSender-Version")))\
                            .countByValue()

for reason, count in count_reason_pingsender.items():
    print("{}:\t{}".format(reason, pct(count, sum(count_reason_pingsender.values()))))
```
    (u'startup', None):	27.5862068966
    (u'shutdown', u'1.0'):	55.1724137931
    (u'shutdown', None):	17.2413793103


Looks like some of these pings are missing the pingsender header from the request. While this is expected for `new-profile` pings with reason `startup`, as they are being sent while Firefox is still active, there might be reasons why this also happens at shutdown:

- that they are not being sent from the pingsender, even though they are generated at shutdown: the pingsender might have failed due to network problems/server problems and Firefox picked them up at the next restart; in this case they would have the same document id;
- that we generated a new-profile ping at shutdown, but failed to mark it as 'generated', and so we received more than one with a different document id.

This leads to other questions:

- How often do we send new-profile pings at shutdown, fail, and then send them again without the pingsender?
- Does that correlate with the duplicates?


```python
newprofile_shutdown_from_bad_clients =\
    subset.filter(lambda p: p.get("payload/reason") == 'shutdown')\
          .filter(lambda p: p.get("clientId") in misbehaving_clients)

newprofile_shutdown_from_good_clients =\
    subset.filter(lambda p: p.get("payload/reason") == 'shutdown')\
          .filter(lambda p: p.get("clientId") not in misbehaving_clients)
```

```python
newprofile_shutdown_from_bad_clients\
    .map(lambda p: p.get("meta/X-PingSender-Version")).countByValue()
```




    defaultdict(int, {None: 10, u'1.0': 32})



This is telling us that most of the shutdown `new-profile` pings are coming from the pingsender, about 76%.


```python
newprofile_shutdown_from_good_clients\
    .map(lambda p: p.get("meta/X-PingSender-Version")).countByValue()
```




    defaultdict(int, {None: 528, u'1.0': 2336})



This is still true with well-behaved clients, as 81% of the same pings are coming with the pingsender. The pingsender doesn't seem to be the issue here: if we generate a ping at shutdown and try to send it with the pingsender, and fail, then it's normal for Firefox to pick it back up and send it. As long as we don't generate a new, different, `new-profile` ping for the same client.

### Does the profileCreationDate match the date we received the pings?


```python
def datetime_from_daysepoch(days_from_epoch):
    return datetime(1970, 1, 1, 0, 0) + timedelta(days=days_from_epoch)

def datetime_from_nanosepoch(nanos_from_epoch):
    return datetime.fromtimestamp(nanos_from_epoch / 1000.0 / 1000.0 / 1000.0)

def get_times(p):
    profile_creation = datetime_from_daysepoch(p["environment/profile/creationDate"])\
                            if p["environment/profile/creationDate"] else None
    ping_creation = datetime_from_nanosepoch(p["meta/creationTimestamp"])
    ping_recv = datetime_from_nanosepoch(p["meta/Timestamp"])
    
    return (p["id"], profile_creation, ping_creation, ping_recv)
    
ping_times = deduped_clientid.map(get_times)
```

```python
ping_creation_delay_days = ping_times.filter(lambda p: p[1] is not None)\
                                     .map(lambda p: abs((p[1].date() - p[2].date()).days)).collect()
```

```python
plt.title("The distribution of the days between the profile creationDate and the 'new-profile' ping creation date")
plt.xlabel("Difference in days")
plt.ylabel("Frequency")

CLIP_DAY = 30
plt.xticks(range(0, CLIP_DAY + 1, 1))
plt.hist(np.clip(ping_creation_delay_days, 0, CLIP_DAY),
         alpha=0.5, bins=50, label="Delays")
```




    (array([  2.84000000e+03,   4.00000000e+01,   0.00000000e+00,
              1.50000000e+01,   0.00000000e+00,   2.70000000e+01,
              2.20000000e+01,   0.00000000e+00,   2.70000000e+01,
              0.00000000e+00,   3.40000000e+01,   3.80000000e+01,
              0.00000000e+00,   3.20000000e+01,   0.00000000e+00,
              3.40000000e+01,   3.40000000e+01,   0.00000000e+00,
              4.00000000e+01,   0.00000000e+00,   2.90000000e+01,
              2.90000000e+01,   0.00000000e+00,   3.40000000e+01,
              0.00000000e+00,   2.00000000e+01,   2.50000000e+01,
              0.00000000e+00,   3.30000000e+01,   0.00000000e+00,
              1.40000000e+01,   2.00000000e+01,   0.00000000e+00,
              1.50000000e+01,   0.00000000e+00,   1.50000000e+01,
              8.00000000e+00,   0.00000000e+00,   8.00000000e+00,
              0.00000000e+00,   4.00000000e+00,   2.00000000e+00,
              0.00000000e+00,   1.00000000e+00,   0.00000000e+00,
              2.00000000e+00,   3.00000000e+00,   0.00000000e+00,
              1.00000000e+00,   3.51000000e+02]),
     array([  0. ,   0.6,   1.2,   1.8,   2.4,   3. ,   3.6,   4.2,   4.8,
              5.4,   6. ,   6.6,   7.2,   7.8,   8.4,   9. ,   9.6,  10.2,
             10.8,  11.4,  12. ,  12.6,  13.2,  13.8,  14.4,  15. ,  15.6,
             16.2,  16.8,  17.4,  18. ,  18.6,  19.2,  19.8,  20.4,  21. ,
             21.6,  22.2,  22.8,  23.4,  24. ,  24.6,  25.2,  25.8,  26.4,
             27. ,  27.6,  28.2,  28.8,  29.4,  30. ]),
     <a list of 50 Patch objects>)





![png](images/output_31_1.png)



```python
np.percentile(np.array(ping_creation_delay_days), [50, 70, 80, 95, 99])
```




    array([    0. ,     0. ,     7. ,   385.6,  1294.2])



The plot shows that most of the creation dates for `new-profile` pings match exactly with the date reported in the environment, `creationDate`. That's good, as this ping should be created very close to the profile creation. The percentile computation confirms that's true for 70% of the `new-profile` pings.

However, things get tricky. The `new-profile` ping was enabled on the 30th of May, 2017 and should only be sent my new profiles. A delay longer than 7 days suggests that either the profile `creationDate` is terribly wrong or that we're sending the `new-profile` ping from existing profiles as well.

### Cross-check the `new-profile` and `main` pings 


```python
main_pings = Dataset.from_source("telemetry") \
                    .where(docType='main') \
                    .where(appUpdateChannel="nightly") \
                    .where(submissionDate=lambda x: "20170531" <= x < "20170607") \
                    .where(appBuildId=lambda x: "20170531" <= x < "20170607") \
                    .records(sc, sample=1.0)
```
    fetching 23348.80037MB in 8027 files...



```python
main_subset = get_pings_properties(main_pings, ["id",
                                                "meta/creationTimestamp",
                                                "meta/Date",
                                                "meta/Timestamp",
                                                "meta/X-PingSender-Version",
                                                "clientId",
                                                "environment/profile/creationDate",
                                                "payload/info/reason",
                                                "payload/info/sessionLength",
                                                "payload/info/subsessionLength",
                                                "payload/info/profileSubsessionCounter",
                                                "payload/info/previousSessionId"])
```
Dedupe by document id and restrict the `main` ping data to the pings from the misbehaving and well behaving clients.


```python
well_behaving_clients =\
    set(subset.filter(lambda p: p.get("clientId") not in misbehaving_clients).map(lambda p: p.get("clientId")).collect())

all_clients = misbehaving_clients + list(well_behaving_clients)
```

```python
main_deduped = dedupe(main_subset.filter(lambda p: p.get("clientId") in all_clients), "id")
main_deduped_count = main_deduped.count()
```
Try to pair each `new-profile` ping with reason `shutdown` to the very first `main` ping with reason `shutdown` received from that client, to make sure that the former were sent at the right time.


```python
first_main = main_deduped.filter(lambda p:\
                                    p.get("payload/info/previousSessionId") == None and\
                                    p.get("payload/info/reason") == "shutdown")
```

```python
newping_shutdown = deduped_docid.filter(lambda p: p.get("payload/reason") == "shutdown")
```

```python
newprofile_plus_main = first_main.union(newping_shutdown)
sorted_per_client = newprofile_plus_main.map(lambda p: (p["clientId"], [(p, p["meta/creationTimestamp"])]))\
                                        .reduceByKey(lambda a, b: sorted(a + b, key=lambda k: k[1]))\
                                        .filter(lambda p: len(p[1]) > 1)\
                                        .map(lambda p: (p[0], [r[0] for r in p[1]]))
num_analysed_clients = sorted_per_client.count()
```

```python
HALF_HOUR_IN_S = 30 * 60

def is_newprofile(p):
    # The 'main' ping has the reason field in 'payload/info/reason'
    return "payload/reason" in p and p.get("payload/reason") in ["startup", "shutdown"]

def validate_newprofile_shutdown(client_data):
    ordered_pings = client_data[1]
    
    newprofile_mask = [is_newprofile(p) for p in ordered_pings]
    
    # Do we have at least a 'new-profile' ping?
    num_newprofile_pings = sum(newprofile_mask)
    if num_newprofile_pings < 1:
        return ("No shutdown 'new-profile' ping found", 1)
    
    # Do we have multiple 'new-profile' pings?
    if num_newprofile_pings > 1:
        return ("Duplicate 'new-profile' ping.", 1)
    
    if not newprofile_mask[0]:
        return ("The 'new-profile' ping is not the first ping", 1)
        
    # If there's a new-profile ping with reason 'shutdown', look for the closest next
    # 'main' ping with reason shutdown.
    for i, p in enumerate(ordered_pings):
        # Skip until we find the 'new-profile' ping.
        if not is_newprofile(p):
            continue

        # We found the 'new-profile' ping. Do we have any other ping
        # after this?
        next_index = i + 1
        if next_index >= len(ordered_pings):
            return ("No more pings after the 'new-profile'", 1)
    
        # Did we schedule the 'new-profile' ping at the right moment?
        next_ping = ordered_pings[next_index]
        if next_ping.get("payload/info/sessionLength") <= HALF_HOUR_IN_S:
            return ("The 'new-profile' ping was correctly scheduled", 1)

        return ("The 'new-profile' ping was scheduled at the wrong time", 1)
    
    return ("Unknown condition", 1)

scheduling_error_counts = sorted_per_client.map(validate_newprofile_shutdown).countByKey()
```

```python
for error, count in scheduling_error_counts.items():
    print("{}:\t{}".format(error, pct(count, num_analysed_clients)))
```
    The 'new-profile' ping is not the first ping:	0.135685210312
    No shutdown 'new-profile' ping found:	0.203527815468
    The 'new-profile' ping was correctly scheduled:	97.8968792402
    The 'new-profile' ping was scheduled at the wrong time:	0.881953867028
    Duplicate 'new-profile' ping.:	0.881953867028


Most of the `new-profile` pings sent at `shutdown`, 97.89%, were correctly generated because the session lasted less than 30 minutes. Only 0.88% were scheduled at the wrong time. The rest of the clients either sent the `new-profile` at startup or we're still waiting for their `main` ping with reason `shutdown`.

### Are we sending `new-profile`/`startup` pings only from sessions > 30 minutes?


```python
newping_startup = deduped_docid.filter(lambda p: p.get("payload/reason") == "startup")
newprofile_start_main = first_main.union(newping_startup)
sorted_per_client = newprofile_start_main.map(lambda p: (p["clientId"], [(p, p["meta/creationTimestamp"])]))\
                                         .reduceByKey(lambda a, b: sorted(a + b, key=lambda k: k[1]))\
                                         .filter(lambda p: len(p[1]) > 1)\
                                         .map(lambda p: (p[0], [r[0] for r in p[1]]))
num_analysed_clients = sorted_per_client.count()
```

```python
def validate_newprofile_startup(client_data):
    ordered_pings = client_data[1]
    
    newprofile_mask = [is_newprofile(p) for p in ordered_pings]
    
    # Do we have at least a 'new-profile' ping?
    num_newprofile_pings = sum(newprofile_mask)
    if num_newprofile_pings < 1:
        return ("No startup 'new-profile' ping found", 1)
    
    # Do we have multiple 'new-profile' pings?
    if num_newprofile_pings > 1:
        return ("Duplicate 'new-profile' ping", 1)
    
    if not newprofile_mask[0]:
        return ("The 'new-profile' ping it's not the first ping", 1)
        
    # If there's a new-profile ping with reason 'startup', look for the closest next
    # 'main' ping with reason shutdown.
    for i, p in enumerate(ordered_pings):
        # Skip until we find the 'new-profile' ping.
        if not is_newprofile(p):
            continue

        # We found the 'new-profile' ping. Do we have any other ping
        # after this?
        next_index = i + 1
        if next_index >= len(ordered_pings):
            return ("No more pings after the 'new-profile'", 1)
    
        # Did we schedule the 'new-profile' ping at the right moment?
        next_ping = ordered_pings[next_index]
        if next_ping.get("payload/info/sessionLength") > HALF_HOUR_IN_S:
            return ("The 'new-profile' ping was correctly scheduled", 1)

        return ("The 'new-profile' ping was scheduled at the wrong time", 1)
    
    return ("Unknown condition", 1)

startup_newprofile_errors = sorted_per_client.map(validate_newprofile_startup).countByKey()
for error, count in startup_newprofile_errors.items():
    print("{}:\t{}".format(error, pct(count, num_analysed_clients)))
```
    The 'new-profile' ping it's not the first ping:	1.41342756184
    The 'new-profile' ping was correctly scheduled:	96.1130742049
    Duplicate 'new-profile' ping:	1.06007067138
    The 'new-profile' ping was scheduled at the wrong time:	0.353356890459
    No startup 'new-profile' ping found:	1.06007067138


The results look good and in line with the previous case of the `new-profile` ping being sent at `shutdown`. The number of times the `new-profile` ping isn't the first generated ping is slightly higher (0.13% vs 1.41%), but this can be explained by the fact that nothing prevents Firefox from sending new pings after Telemetry starts up (60s into the Firefox startup some addon is installed), while the `new-profile` ping is strictly scheduled 30 minutes after the startup.

### Did we receive any `crash` ping from bad-behaved clients?
If that happened close to when we generated a `new-profile` ping, it could hint at some correlation between crashes and the duplicates per client id.


```python
crash_pings = Dataset.from_source("telemetry") \
                     .where(docType='crash') \
                     .where(appUpdateChannel="nightly") \
                     .where(submissionDate=lambda x: "20170531" <= x < "20170607") \
                     .where(appBuildId=lambda x: "20170531" <= x < "20170607") \
                     .records(sc, sample=1.0)
```
    fetching 356.90416MB in 3290 files...


Restrict the crashes to a set of useful fields, just for the misbehaving clients, and dedupe them by document id.


```python
crash_subset = get_pings_properties(crash_pings, ["id",
                                                  "meta/creationTimestamp",
                                                  "meta/Date",
                                                  "meta/Timestamp",
                                                  "meta/X-PingSender-Version",
                                                  "clientId",
                                                  "environment/profile/creationDate",
                                                  "payload/crashDate",
                                                  "payload/crashTime",
                                                  "payload/processType",
                                                  "payload/sessionId"])
crashes_misbehaving_clients = dedupe(crash_subset.filter(lambda p:\
                                                             p.get("clientId") in misbehaving_clients and\
                                                             p.get("payload/processType") == 'main'), "id")
newprofile_bad_clients = subset.filter(lambda p: p.get("clientId") in misbehaving_clients)
```

```python
def get_ping_type(p):
    return "crash" if "payload/crashDate" in p else "new-profile"

newprofile_and_crashes = crashes_misbehaving_clients.union(newprofile_bad_clients)

# Builds an RDD with (<client id>, [<ordered reason>, <ordered reason>, ...])
joint_ordered_pings = newprofile_and_crashes\
                        .map(lambda p: (p["clientId"], [(get_ping_type(p), p["meta/creationTimestamp"])]))\
                        .reduceByKey(lambda a, b: sorted(a + b, key=lambda k: k[1]))\
                        .map(lambda p: (p[0], [r[0] for r in p[1]]))

sorted(joint_ordered_pings.map(lambda p: tuple(p[1])).countByValue().items(), key=lambda k: k[1], reverse=True)
```




    [(('new-profile', 'new-profile'), 15),
     (('new-profile', 'new-profile', 'new-profile'), 4),
     (('new-profile',
       'crash',
       'crash',
       'new-profile',
       'crash',
       'new-profile',
       'new-profile',
       'crash',
       'new-profile',
       'crash',
       'new-profile'),
      1),
     (('new-profile', 'crash', 'new-profile'), 1),
     (('new-profile',
       'new-profile',
       'new-profile',
       'new-profile',
       'new-profile',
       'crash',
       'crash',
       'crash',
       'crash',
       'crash',
       'new-profile'),
      1),
     (('new-profile', 'new-profile', 'crash'), 1)]



The first groups of reported ping sequences, don't contain any `crash` ping and account for most of the `new-profile` duplicates pattern. The other sequences interleave `new-profile` and main process `crash` pings, suggesting that crashes might play a role in per-client duplicates. However, we only have crashes for 3 clients over 23 misbehaving ones, not enough to draw conclusions from.


```python

```
