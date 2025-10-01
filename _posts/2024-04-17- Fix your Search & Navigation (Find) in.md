# Fix your Search & Navigation (Find) indexing job, please — Quan Mai’s blog

![Quan Mai](https://miro.medium.com/v2/da:true/resize:fill:40:40/0*CwmFQL3u3gyeLpv2)

**Author:** [Quan Mai](https://medium.com/@vimvq1987?source=post_page---byline--8b800a802c19---------------------------------------)  
**Published:** Apr 17, 2024  
**Reading time:** 3 min

---

Once upon a time, a colleague asked me to look into a customer database with weird spikes in database log usage. (You might start to wonder why I am always the one who looks into weird things, is there a pattern here)

Upon reviewing the query store, I noticed a very high logical reads related to `tblScheduledItem`. From past experience, it was likely because of fragmentation of indexes in this table (which has only one clustered). I did a quick look at the table, and confirmed the index indeed has high fragmentation. I suggested to do a rebuild of that index and see what happen. Well, it could have been one of the daily simple quick questions, and I almost forgot about it.

A few days passed, the colleague pinged me again. Apparently they rebuilt it but it does not really help. That raised my eyebrows a little bit, so I dug deeper.

To my surprise, the problem was not really fragmentation (it definitely contributed). The problem is that the column has a column of type `nvarchar(max)`, and it's for recording the last text from the `Execute` method of the scheduled job. It was meant for something short like "The job failed successfully", or "Deleted 12345 versions from 1337 contents". But because it's `nvarchar(max)` it could be very, very long. You can, in theory, store the entire book content of a few very big libraries in there.

Of course because you can, does not mean you should. When your column is long, each read from the table will be a burden to SQL Server. And the offending job was nothing less than our S&N indexing job.

In theory, any job could have caused this issue, however it’s much more likely to happen with the S&N indexing job for a reason — it keeps track of every exception thrown during the indexing process, and because it indexes each and every content of your website (except the ones you specifically, explicitly tell it not to), the chance of its running into a recurring issue that affects multiple (reads, a lot) of content is much higher than any built-in job.

I asked, this time, to trim the column, and most importantly, fix any exceptions that might be thrown during the index. I was on my day off when my colleague notified me that the job is running for 10h without errors, as they fixed it. Curious, so I did check some statistics. Well, let those screenshots speak for themselves:

![Screenshot 1](https://miro.medium.com/v2/resize:fit:809/0*MRZqUDcPNiBFsXNL)

![Screenshot 2](https://miro.medium.com/v2/resize:fit:574/0*aIW19Na3Te3BKfw8)

The query itself went from 16,000ms to a mere 2.27ms. Even better, each call to get the list of scheduled jobs before resulted in 3.5GB logical reads. Now? 100KB. A lot of resource saved!

So, make sure your job is not throwing a lot of errors. And fix your S&N indexing job.

**P/S:** I do think the S&N indexing job should have a simpler return result. Maybe “Indexed 100.000 content with 1234 errors”, and the exceptions could have been logged. But that’s debatable. For now, you can do your part!