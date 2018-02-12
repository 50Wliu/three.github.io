---
layout: post
title: Reporting Misconfigured IAM permissions in GoGuardian
    management software
published: false
---

*This post is co-authored by Winston Liu and Eric Roberts, with the first-person
point of view being Eric's. We jointly discovered and reported two security issues
in GoGuardian's Chromebook management software, and were rewarded by the company
for our work. This post chronicles what happened.*

First, a confession: this is about a bug found and submitted almost two years
ago. We always had it in the back of my mind to write this post but laziness and
procrastination led us to putting it off until now. This post has been compiled
from our memory of the events along with chats and emails recovered from Google
Takeout. Hopefully people still find this useful or entertaining.

# Background

At the time of these events we were seniors at Morristown High School, a
moderately-sized public high school primarily serving Morristown, Morris
Township, and Morris Plains in Morris County, New Jersey. Morris School District
was just starting its Chromebook Initiative to "cultivate and support learning
that reflects contemporary exchanges and interactions[\*][0]." To us, this meant
that every student in the high school received a Chromebook (typically a Dell
Chromebook 11 or Samsung Series 3 Chromebook) managed by the high school.

[0]: https://www.morrisschooldistrict.org/domain/215

Our interest in discovering vulnerabilities in the Chromebooks followed rumors
that someone had already discovered issues in the software. I never heard
specific details and never confirmed the rumors. True or not, they peaked our
interest and led us to further investigate the nature of the software.

# Discovery

The Chromebooks given to students were set up as managed devices owned by the
district. Access to the Developer Tools were disabled, Chromebooks were
automatically updated whenever a new Chrome OS version came out, and the
GoGuardian monitoring software was preinstalled on all Chromebooks. Despite
these restrictions, we could still view installed extensions, install anything
from the Chrome Web Store and prepend `view-source:` to any URL or path to get
the source.

We don't recall all the individual steps we took or red herrings we followed
before finding the two issues we would eventually report. It took about a week
of trial and error both in class and after practice before finding everything.
Since these wrong turns are not recorded in the chat, what's here is a far more
streamlined version of what actually happened.

Our investigation started with looking for the source of the management
software. It was quite obvious that there was such software installed on the
Chromebooks, as attempting to access certain sites (such as the beloved
[slither.io][1]) were blocked locally. Furthermore, some of the more
technologically-savvy teachers had begun to utilize the classroom-facing
features of the software, and it is there that we learned that the software was
developed by GoGuardian. However, there were no extensions proudly labelled
"GoGuardian" or "Chromebook Management" so we focused on the two extensions that
came preinstalled with each Chromebook. Hidden in the minified source of one of
these extensions was a call to the AWS javascript client which included an
`accessKeyId` and `secretAccessKey`. This seemed like a promising start, so we
tried to find looking for what these keys had access to.

[1]: http://slither.io/

Setting up the AWS cli and plugging in the discovered IAM keys revealed two
services we had access to: dynamodb and kinesis. We only had readonly access to
dynamodb and the data looked uninteresting so we moved on to kinesis. Although
it took some reading and fiddling trying to figure out what kinesis actually did
and how to use it we eventually found we had the ability to read from streams of
student data spread across several so-called "shards." The cli proved
ineffective at sampling the data passing through the shards so I wrote the
following script to assist us (apologies for code quality).

```python
import boto3
import json
import sys

strName = sys.argv[1]

client = boto3.client("kinesis");

response0 = client.describe_stream(
        StreamName=strName
        )
print("Found an "+response0["StreamDescription"]["StreamStatus"]+" stream!")
print("Stream has "+str(len(response0["StreamDescription"]["Shards"]))+" shards.")
print("Retention Period: "+str(response0["StreamDescription"]["RetentionPeriodHours"])+" hours.")
for s in response0["StreamDescription"]["Shards"]:
    print("  Shard: "+s["ShardId"])
    response1 = client.get_shard_iterator(
            StreamName=strName,
            ShardId=s["ShardId"],
            ShardIteratorType="LATEST",
            )
    shard = response1["ShardIterator"];
    response2 = client.get_records(
            ShardIterator=shard,
            Limit=1,
            )
    records = response2["Records"]
    if ( len(records) < 1 ):
        print("No records in shard.")
        continue
    data = records[0]["Data"]
    print( data )
```

Among the output was JSON-formatted data clearly sent from student Chromebooks across
the country announcing its user had accessed a site with a blacklisted word. At this
point we decided to look for someone to contact at GoGuardian, although in the
time it took to send the email we managed to find something else more
interesting.

Although knowing which students are visiting naughty pages is somewhat
interesting, we were convinced GoGuardian was collecting more detailed
information from students. We had already found debugging the extension on the
Chromebooks was a lost cause so I decided to try my luck debugging the software
on a personal computer. However, the management extensions and restrictions,
discussed earlier, are loaded into Chrome after associating a user profile with
the school Google account. This meant that all the policies were also active on
our personal devices. Winston managed to circumvent the devtools restrictions
through the `--remote-debugging-port` switch, which opens up the managed Chrome
instance to debugging from another Chrome instance's dev tools. By starting up
a personal Chrome instance to do just that, we hit the second roadblock: the
district had configured the GoGuardian software to disable itself when not
running on a Chromebook. After looking through the pretty-printed code we
determined that it was simply checking for a Chrome OS user agent, and so we
were able to trick GoGuardian by spoofing our user-agent through another
command-line switch, `--user-agent`.

After successfully enabling the extension and looking at its network activity
through the 'remote' Chrome instance, we saw GoGuardian occasionally uploading
screenshots to an S3 bucket. These screenshots revealed the other half of what
we had found in the kinesis shards: whenever someone visits a page with
blacklisted words, the page URL is sent to kinesis while a screenshot of the
page is uploaded to S3. The URL was random and there was far too much entropy
to randomly guess other screenshots, but Winston discovered that crucially, the
root path of the bucket was unsecured and had a listing of all the screenshots
collected by GoGuardian. Furthermore, the screenshots could be filtered by
query parameters, such as by organization or even by user. Due to the amount of
personally-identifiable information freely searchable in the bucket we contacted
GoGuardian with both issues the next day.

# Reporting

GoGuardian, at the time we wanted to report everything, did not have a bug
bounty program. It took a couple tries to find someone who responded but
eventually our email got through.

I don't have a full timeline because GoGuardian fixed some of the issues before
we got a reply, so the dates on the emails make it seem like things took longer
to fix than they did.

> Date: Thu, 18 Feb 2016 11:28:03 -0500
>
> Subject: GoGuardian Security Vulnerability
>
> From: Eric Roberts
>
> To: Advait Shinde
>
> Cc: Winston Liu
>
> To the developers of GoGuardian,
>
> We are students at a high school that uses GoGuardian monitoring software to
> manage Google Chromebooks. We believe we have found a security vulnerability
> allowing unauthorized persons to access private student data. This information
> includes browsing history and snapper screenshots and we believe an attacker
> could use this to access more. ec These vulnerabilities involve exclusively
> the full read access given to certain AWS data. Using the IAM authorization
> keys from bundle.js file in the Chromium M extension read access is given to
> DynamoDB and Kinesis, allowing read access to user data. Additionally using
> the REST API for S3 storage it is possible to list and access snapper
> screenshots of student Chromebooks.
>
> As a monitoring software company we know you value the privacy of the students
> that use your software as much as the students expect to have the detailed
> usage history you collect kept in a private and secure manner.
>
> If the information in this email is not sufficient to provide a fix, please
> respond with any questions. Please understand that we did not do anything with
> malicious intention and made sure to report all our findings at the soonest
> time appropriate (with this email). We ask you to accept this correspondence
> as a helpful gesture to better your services.
>
> Sincerely, Eric Roberts Winston Liu
>
> Students of Morristown High School, Morris School District

> Date: Fri, 26 Feb 2016 11:35:35 -0800
>
> Subject: Re: GoGuardian Security Vulnerability
>
> From: Advait Shinde
>
> Hey Eric. Thanks for the warning! We are escalating here internally and
> getting this resolved ASAP. [Moved to bcc]

> Date: Fri, 26 Feb 2016 12:00:41 -0800
>
> Subject: Re: GoGuardian Security Vulnerability
>
> From: Advait Shinde
>
> To: Eric Roberts, Winston Liu
>
> Hey Winston. Just wanted to let you know we're on top of this - it's a P0
> security priority for us.
>
> We're in the process of doing a full audit of all of our IaM roles and
> checking cloudtrail to determine if there was any unauthorized access with
> this key.
>
> We will keep you updated.

> Date: Tue, 15 Mar 2016 09:15:00 -0400
>
> Subject: Re: GoGuardian Security Vulnerability
>
> From: Winston Liu
>
> To: Advait Shinde
>
> Cc: Eric Roberts
>
> Hello Advait,
>
> A few weeks ago, I bumped into an old friend who's now in college. We talked
> for a bit, and he happened to ask if the school district was still using
> GoGuardian (he's extremely paranoid regarding security). As we had more free
> time than usual that week, Eric and I decided to see what GoGuardian could
> access as a fun exercise. After we found the IAM key in the bundle.js file we
> used the AWS CLI to try various services, expecting the key to be write-only.
> To our surprise, the key had read access to much of the student data. After
> confirming this was, in fact, a vulnerability and not intended behavior, we
> contacted you to fix the problem.
>
> In addition, although the IAM key has been made read-only, there is still one
> more vulnerability: the S3 storage for Snapper snapshots is publicly available
> online at https://s3-us-west-2.amazonaws.com/goguardian-snapper-production/.
> All snapshots taken by Snapper can be accessed by simply appending the key to
> the end of the URL. In addition, the results can be filtered through query
> parameters, like so: redacted
>
> The main page should return a 403 error to prevent that.
>
> From,
>
> Winston and Eric

> Date: Mon, 21 Mar 2016 10:00:26 -0700
>
> From: Advait Shinde
>
> To: Winston Liu
>
> Cc: Eric Roberts
>
> Hey Winston and Eric!
>
> We're doing a full audit of all of our bucket permissions right now.
> Regarding the snapper bucket: the keys themselves were designed to have enough
> entropy such that auth was not needed. However, as you guys discovered, the
> list operation was available, effectively undermining the opaque-ness of the
> keys. This should no longer be the case. We are currently working on a service
> to wrap the s3 bucket to provide auth.
>
> We are thrilled that you guys were able to find these vulnerabilities!  Thank
> you so much for your help. By my record, you guys discovered two
> vulnerabilities (permissions for the IAM key, permissions for the S3 bucket).
> I am in the process of setting up a bug bounty program to reward you for your
> efforts.
>
> Cheers,
>
> Advait

# Summary

While we were seniors at Morristown High School, we discovered and reported
two security vulnerabilities in the GoGuardian management software,
both of which were exposing personally-identifiable information to the public.

# Reward

For our efforts we each received a box of GoGuardian swag and a $250 Visa Gift
Card. At first we were scared of reporting since GoGuardian did not have a bug
bounty program, but we found they were happy to take our report and were helpful
in providing updates.
