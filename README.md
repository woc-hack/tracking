# The secret life of hackathon code

## Problem

Developing and showcasing a functioning (software) prototype is at the core of most hackathon events. Our understanding of whether and how they utilize existing code and what happens to the code that has been developed during a hackathon after the event has ended is however still limited. In particular the connection between utilizing existing code, developing code during a hackathon and the continued use of that code after an event is still unknown. The aim of this project is to close this gap.
Breaking this down to individual research questions would amount to something like:

  - RQ1: Where does hackathon code come from?
     - RQ1.a: Was it created during the hackathon? (i.e. in the same repository that was used during the hackathon at the time when the hackathon took place)
     - RQ1.b: Was it created by the same individuals but in a different context?
     - RQ1.c: Did they reuse existing code that was not theirs?

  - RQ2: What happens to hackathon code after the event has ended?
     - RQ2.a: Does it only live on among the same people?
     - RQ2.b: Do projects merge?
     - RQ2.c: Does the code get reused in a (larger) open source project?

  - RQ3: Which factors influence continuation? (The Devpost dataset contains additional context information we could leverage such as which teams won, how large the teams were, how large the hackathon was, etc.)


## Approach

- Identify blobs created by hackathon projects p2c | c2b
- Determine the first commit for each blob (b2a)
- If it is in the hackathon project look at the other commits (b2c) and determine projects for them (c2p)
     - these ather projects are where the code moved to
 - otherwise look at the prior commits and determine prjojects
     - that is where the code came from
     
     
```
cd /data/play/trackhacks
#get hackathon projects
cut -d, -f3 woc-commits-authors.csv | lsort 1G -u > ps

# pmap p to P
cat ps | ~/lookup/getValues p2P | cut -d\; -f2 | lsort 1G -u > Ps

# get commits
cat Ps | ~/lookup/getValues -f P2c | cut -d\; -f2 | lsort 10G -u > Cs

# get blobs
cat Cs | ~/lookup/getValues -f c2b | cut -d\; -f2 | lsort 10G -u > Bs
# total number of blobs created by these commits:
wc Bs
  5080658 

# get the first commit for each blob 
cat Bs | ~/lookup/getValues b2a | gzip > B2as

# order these commits by time (some are from others not from hackathon)
zcat B2as | cut -d\; -f4 | lsort 1G -u > CsBO

# select only commits from the hackathon projects
join Cs CsBO | gzip > CsBO.j

# Now these are all the blobs created by these commits
zcat B2as | ~/bin/grepField.perl CsBO.j 4 | cut -d\; -f1 | gzip > BOs
# the blobs that came from somwhere else are:  
cat Bs | lsort 10G | join -v1 - <(zcat BOs) | gzip > BNs
```
Out of 5080658 blobs in hackathon repos only 2944578 have originated there: the rest (2136080) 
were created somwhere else:
```
zcat BOs | wc -l
2944578
zcat BNs | wc -l
2136080
```

Now identify projects and commits that used the blobs originated during hackathons
somwhere else (and after the hackathons)
```
# find projects for all the commits for the blobs originated during hackathons 
zcat c2BOta.s| cut -d\; -f1 | uniq | ~/lookup/getValues -f c2P | gzip > c2BOta2P.s
# exclude hackathon projects
zcat c2BOta2P.s | ~/lookup/grepFieldv.perl Ps.gz 2 | gzip > c2BOta2Pnew.s
#finally commits, times, and projects that used blobs originated during hackathons
zcat c2BOta.s | lsort 20G -t\; -k1 | join -t\; - <(zcat c2BOta2Pnew.s| lsort 10G -t\; -k1,2) | gzip > c2BOtaP.s
# Count projects
zcat c2BOtaP.s|cut -d\; -f4 | lsort 1G -u | wc -l
374140
```

Finally, identify projects and commits that provided the blobs used during hackathons:
```
# Get projects for each commit
zcat c2BNta.s| cut -d\; -f1 | uniq | ~/lookup/getValues -f c2P | gzip > c2BNta2P.s
# join to creat commit, blob, time, author, project
zcat c2BNta.s | lsort 20G -t\; -k1 | join -t\; - <(zcat c2BNta2P.s| lsort 10G -t\; -k1,2) | gzip > c2BNtaP.s
# Count projects
zcat c2BNtaP.s|cut -d\; -f4 | lsort 1G -u | wc -l
1619565

# lets try to get rid of the widely used blobs (probably templates/licenses/trivial files) 
# first sort by blob to make it easy to count commits for each blob
zcat c2BNta.s | lsort 5G -t\; -k2 | gzip > c2BNta.sb

# count blobs for each commit
zcat c2BNta.sb | cut -d\; -f2 | uniq -c > c2BNta.sb.c
# select blobs that have under 100 commits (unusal ones)
awk '{ if($1<100) print $2}' c2BNta.sb.c | gzip > unusualBlobs
# filter to include only unusualBlobs
zcat c2BNta.s | ~/lookup/grepField.perl unusualBlobs 2 | gzip > c2BNta.s1
# join to creat commit, blob, time, author, project
zcat c2BNta.s1 | lsort 20G -t\; -k1 | join -t\; - <(zcat c2BNta2P.s| lsort 10G -t\; -k1,2) | gzip > c2BNtaP.s1 
# Count projects for unusual blobs only
zcat c2BNtaP.s1|cut -d\; -f4 | lsort 1G -u | wc -l
178308
```
If we exclude common blobs (probably templates/licenses/trivial
files), the number of projects where hackathons sourced their code
is 178308. If we include common blobs, it is almost ten times larger:1619565

For comparison, the number of projects using hackathon-originated
files is 374140.

In summary, hackathon repositories extensively reuse files from existing
repositories and the files created in the hackathon are also used
commonly used later in other repositories. 

## Current status (reverse chronological):

> A few clarifications:
> Do you mean https://devpost.com has 16497 repositories?

No. Devpost has much more than that. From the data dump I created in May 2018 there were 16497 repositories that were publicly available and that show some activity. Since then Devpost has changed their policy and does not allow crawling anymore so getting a newer version of the dataset might be desirable but difficult.

> When you say
> "I was ableto identify 1592896  unique blobs and 562955 commits in 16497 repositories."
> do you mean: p2c for these 16497  repos gives 562955 commits and c2b for these commits gives 1592896  unique blobs?

Partly. I used P2c to get the commits and P2b to get the blobs.

> When you say:"40009 (2.5 %) were created in this repository"  do you mean that
> there was no commit in 16497 repositories that created 97.5% of the blobs?

No. I mean only 40009 blobs had their ORIGINAL commit in one of the these 16497 repositories. All mentioned repositories have commits.

> Perhaps you have data (e.g., list of projects) on woc servers, so I can look at it directly and comment more precisely?

There are 2 files in /data/play/trackhacks that contain the information that I gathered so far:
 - woc-blobs-authors.csv contains the result of P2b and b2a
 - woc-blobs-commits.csv contains the result of P2c and c2ta
 
 
 
As discussed with you individually I have started to collect data from WoC based on Github repositories that I know are connected to hackathons based on the database Devpost (https://devpost.com/). So far I was able to identify 1592896 unique blobs and 562955 commits in 16497 repositories.

Comparing the commit IDs from the blobs to the commit IDs in the commits I found that only 40009 (2.5 %) were created in this repository which I find quite astonishing. I am not entirely sure though if this comparison is fair / accurate since there are quite a lot of commits in those projects that have a different ID but are done by the same committer. So maybe I am doing something wrong there.

I do however think that this project could be great showcase of WoC in general as well since this kind of analysis is definitely not possible with any other kind of resource. So maybe we could enlist help from a student, programmer or post-doc to take this further. I started searching here in Tartu but it would be even better if there would be someone from Tennessee that could help since there are other people that could help figure out the intricacies of WoC better than I could do here. I would be happy to help with supervising / providing additional data from the database though.


 
