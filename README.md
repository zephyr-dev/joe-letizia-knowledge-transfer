# Group Document Export
The group document exporter is a feature that takes all documents that an investor group may be interested in and exports them to a single, organized, S3 bucket. I wrote this for Kellen when we were losing a lot of customers to ProSeeder so that we could give those customers their data. I meant to embed this in the admin panel of Zephyr, but I never got around to it. Plus, given its nature of being distributed across many background jobs, I wasn't sure how to check for completion, and what the feedback should be on completion. 

The bulk of the code can be found in the `app/models/investor_groups/document_export` folder. To invoke the exporter, call: 
```
InvestorGroups::DocumentsExport::Orchestrator.new(investor_group).export(s3_bucket_name)
```
The best way to call it is from the Rails console on production. Many background jobs will be enqueued to the `externally_enqueued` resque queue. __Make sure to spin up workers after enqueuing the jobs.__ Depending on the size of the group, the documents could end up taking up several GB on S3. 

After the background jobs are all complete, __make sure to ramp the worker count back down__ as not to pay for workers we don't need. 

To get the documents to Kellen, I generally sync'd the bucket to my local machine using the aws-cli tool. You can find the documentation [here](http://docs.aws.amazon.com/cli/latest/reference/s3/sync.html). I would then zip that local folder and upload that zip to google drive and share it with Kellen. You COULD make the s3 folder public, but that's a bit risky.

# Group Data Export
The group data export is similar to the group document export in that it was written for Kellen when we had many customers leaving the platform. This export generates a series of CSV files that contain the investor group's data. 
To invoke the data exporter, go to `https://gust.com/admin/reports/investor_group_data_extract` as a Gust admin. You will be prompted to select an investor group from the drop down. Submit, and a report will be emailed to you when it is completed. __NOTE:__ If the group you are exporting data for is particularly large (such as Sand Hill Angels or perhaps New York Angels), the background worker jobs on Heroku might fail becuase they don't have enough memory. If you run into this, you can invoke these jobs from the local rails console and set your DB url to that of the production DB, or you can use higher end workers for the background workers (__make sure to set them back afterwards__).

The data export code sits in the `DataExports` namespace. The first class in the hierarchy is `DataExports::InvestorGroups::FullExtract`. This class can be instantiated with and investor group, and invoking the `execute` method with a recipient email will kick off a chain of dependencies that will generate the data and corresponding CSV files. 

To invoke, run the following from the rails console: 
```
DataExports::InvestorGroups::FullExtract.new(investor_group).execute(recipient_user)
```

# Hiring

Nash has a good handle on hiring, but when I was running the process, I was trying a few new things. For the past few hiring pushes we made, I made different candidates do different things based on how they got into our funnel. Our hiring sources can be thought of as such:
1. Resume submission on the web site
2. Third party recruiter
3. Personal recomendation

For those candidates coming in through third party recruiters and personal recommendation, I generally scheduled a call with them to talk about what we were looking for and what they were looking for. Based on how that call went, I would have them complete the tech screen with a developer. For cold resume drops on the web site, I was beginning to send the vast majority of them a code challenge (build a set) before investing serious amounts of time. This was because about 90% of those who apply on our website are spam. They either never get back to you, don't have the necessary experience, or don't even live (nor want to move) in the USA. I felt better with this method becuase I felt that it was easy to begin to develop biases. Sending a them a code challenge allows them to reject themselves. 

I encourage the next person who handles hiring to be dilligent about the challenge they select. Something too hard can be off putting to legitimate candidates. This is esspecially true for senior candidates. However, we've realistically never hired a candidate from a resume; all of our hires have been through personal recommendations/connections, career fairs, or third party recruiters.

The coding challenge I have been using has simply been the set implementation we have used when the tech screen is administered 1-on-1 with a developer. You can find a default email reply containing an attachment describing the challenge to the candidate on workable labeled `Take home tech screen (set)`.

# NT Activity Summary

The NT Activity Summary is meant to be an entrepreneur facing Daily Recap. It pushes event data to cassandra, where that data is later retrieved from, aggregated, and used to produce an email that will be sent to the NT weekly. 

The infrastructure for the NT Activity Summary is in place. Dean and I worked on it for a few weeks. It follows a pub-sub model; events are published and subscribers enqueue background jobs to write that event data to cassandra. On the other end, a job can be set up to periodically run so that the data is aggregated and sent to the Entrepreneur. 

Originally, the activity summary was going to be sent to the NT and have the events broken up into batches by the investor groups that committed them. For example, we originally intended to have sections for each investor group, and then summarize how many people viewed the NT's company profile, or played their video, or viewed their pitch deck. We soon realized taht this would be difficult because if the investor looks at the company's pitch deck via their company profile rather than the deal room, there would be no sure-fire way to group that investor to a specific angel group. To combat this, we are simply grouping by the activity itself; "3 investors looked at your pitch deck" or "5 investors looked at your company profile."
