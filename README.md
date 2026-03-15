FreshCart Data Lake — Presentation Script
BADM 558 | Team E5 | 10-Minute Video
VP of Engineering Stakeholder | Option B + Phase 2 Theory

MEMBER SPLIT
SectionTimeSpeakerIntro + what we built0:00 – 1:00Member 1Architecture walkthrough1:00 – 2:30Member 1Lambda ETL — cleaning rules2:30 – 4:00Member 2Scale problem — theory + implications4:00 – 5:45Member 2Batch processor fix5:45 – 7:00Member 1Live demo — queries + RDS verification7:00 – 8:30Member 2Design decisions + stakeholder connection8:30 – 9:30Member 1Close9:30 – 10:00Member 1

SCRIPT

INTRO (0:00 – 1:00) — Member 1
[Screen: Architecture diagram]
"Hi, we're Team E5 from BADM 558. This is our Mini Project 1 — Building a Data Lake at Scale for FreshCart, a grocery delivery company with 50 stores across Illinois.
FreshCart generates one messy order file per store per day. Our job was to build their first data lake on AWS — ingest 500 files, clean the data, make it queryable, and figure out why naive approaches break at this scale.
We'll walk through our architecture, our ETL logic, the scale failure we analyzed, how we fixed it, and a live demo of the working system."

ARCHITECTURE WALKTHROUGH (1:00 – 2:30) — Member 1
[Screen: Keep architecture diagram visible, point to each component as you name it]
"Our pipeline has five AWS services.
S3 raw zone — this is where all 500 order CSVs and the customer feedback JSON land. Files sit under a raw/ prefix organized by store and date.
Lambda is our ETL layer. We built two functions: an event-triggered Lambda for Phase 1 small-scale testing, and a batch processor for the full 500-file run. I'll explain why we needed both.
S3 processed zone stores the cleaned output — 500 cleaned CSVs and the cleaned feedback JSON, each prefixed with cleaned_.
RDS MySQL is our query layer. Three tables: clean_orders for processed order data, data_quality_log for every issue we found during cleaning, and customer_feedback for the JSON data.
CloudWatch captures all Lambda execution logs. And CloudShell is how we ran the data generation script and executed the six business queries against RDS.
The key design choice in this architecture is that Lambda writes to both S3 and RDS — S3 for durability and replayability, RDS for queryability. These serve different purposes and we wanted both."

LAMBDA ETL — CLEANING RULES (2:30 – 4:00) — Member 2
[Screen: Switch to Lambda console or show the lambda_function.py code — highlight clean_data function]
"Let me walk through what the Lambda actually does.
The first thing it does is detect the file format from the extension. CSV files go through the order cleaning pipeline. JSON files go through a separate feedback validation path. This is the multi-format ingestion requirement.
For CSV order files, we apply six rules.
Duplicates — same order_id appearing twice from checkout retries. We keep the first occurrence and log the duplicate. Dropping silently would hide the issue from the operations team.
Missing customer_id — guest checkout orders with no customer attribution. We drop these because they have no value for customer analysis. You can't segment what you can't attribute.
Negative or zero quantity — system glitches producing invalid line items. We drop these because total_amount equals quantity times unit_price — a negative quantity would corrupt every revenue query.
Inconsistent date formats — different POS versions write dates as 01/15/2026, or 15-Jan-2026, or 2026/01/15. The underlying date is valid, just formatted differently. We standardize everything to YYYY-MM-DD using pandas. This one we fix rather than drop.
Missing payment method — the order identity and revenue data are intact, only one field is missing. We fill with 'unknown' to preserve the row. Dropping would hide valid sales from revenue analysis.
Invalid products — name is N/A or category is UNKNOWN. System glitch with no recoverable data. These would appear as an N/A category in our revenue-by-category query, so we drop them.
Every dropped or fixed record is written to data_quality_log with the source file, order ID, issue type, and a description. The operations team has a full audit trail."

SCALE PROBLEM — THEORY AND IMPLICATIONS (4:00 – 5:45) — Member 2
[Screen: Show CloudWatch or architecture diagram — highlight the Lambda-to-RDS connection]
"Now the most important part — what happens when you go from 3 files to 500, and why the obvious architecture breaks.
We took Option B — we deployed the batch processor directly rather than running the stress test. But we analyzed the failure mode in detail before making that decision, and I want to explain exactly what would have happened.
If we had activated the S3 event trigger before pulling all 500 files, here's the cascade:
S3 fires one Lambda invocation per file upload. At 500 files arriving simultaneously, Lambda spins up approximately 500 concurrent execution environments. Each one calls pymysql.connect — opening a new TCP connection to RDS.
Our RDS instance is a db.t3.micro with 1 gigabyte of RAM. MySQL calculates max_connections from available memory — that works out to approximately 66 concurrent connections. So the first 66 Lambda functions connect and begin processing. The remaining 434 get 'Too many connections' from MySQL and fail.
Here's where it gets worse: Lambda's built-in retry logic automatically re-invokes failed functions up to twice. So now we have a second and third wave of connection attempts on top of connections still held open by the first wave. We're not recovering — we're making it worse.
The result in RDS would be unreliable in both directions. Some files processed multiple times — duplicates. Other files silently skipped — missing data. Any business query run against this state would produce wrong answers with no obvious signal that the data is corrupted.
We chose not to run this test because fixing corrupted RDS data mid-project — truncating tables, re-running everything — would have cost us time we didn't have. But more importantly, the failure is deterministic. The math is: 500 invocations, 66 connection slots, guaranteed failure. We didn't need to observe it to understand it."

THE FIX — BATCH PROCESSOR (5:45 – 7:00) — Member 1
[Screen: Show freshcart_batch_processor.py — highlight the single connection at top of handler]
"The batch processor solves this with a fundamentally different execution model.
Instead of one invocation per file, we have one invocation for all 500 files. We invoke it manually with a single JSON event.
The critical line is here — pymysql.connect is called exactly once at function startup. That single connection object is passed into every file's processing loop. RDS never sees more than one active connection. The ceiling problem disappears entirely.
Files are processed sequentially in a loop. Each file completes its full pipeline — read from S3, clean, write to processed zone, insert to RDS — before the next file begins.
We also added an action parameter so we can run just the JSON file, just the CSVs, or everything. This was useful for testing — we processed and verified the feedback file independently before running the full 500-file batch.
The tradeoff is speed. Sequential is slower than parallel. But for FreshCart's overnight batch pattern, correctness matters more than throughput. The full 500-file run completed within the 15-minute Lambda timeout."

LIVE DEMO (7:00 – 8:30) — Member 2
[Screen: Switch to AWS console — have all tabs open before recording]
"Let me show you the working pipeline.
[Show S3 raw bucket — navigate to raw/ prefix]
Here's our raw bucket. You can see the 500 order files and the customer_feedback.json under the raw/ prefix.
[Show S3 processed bucket — navigate to processed/ prefix]
And the processed bucket — 500 cleaned CSVs and the cleaned feedback JSON. Every file prefixed with cleaned_.
[Switch to CloudShell — connect to RDS]
Now let's verify the data in RDS.
mysql -h mp1-558-teame5-rds.csxdborekups.us-east-1.rds.amazonaws.com -u admin123 -p
USE freshcart;
[Run Query 1]
SELECT COUNT(*) AS total_clean_orders FROM clean_orders;
— We loaded [X] clean order rows from 500 files.
[Run Query 2]
SELECT category, COUNT(*) AS order_count, ROUND(SUM(total_amount), 2) AS total_revenue
FROM clean_orders GROUP BY category ORDER BY total_revenue DESC;
— [Name top category] is our highest revenue category.
[Run Query 5]
SELECT issue_type, COUNT(*) AS issue_count FROM data_quality_log
GROUP BY issue_type ORDER BY issue_count DESC;
— You can see all six issue types logged — duplicates, missing customers, invalid quantities, and invalid products.
[Run feedback query]
SELECT COUNT(*) FROM customer_feedback;
SELECT * FROM customer_feedback LIMIT 5;
— [X] feedback records loaded from the JSON file after cleaning."

DESIGN DECISIONS + STAKEHOLDER CONNECTION (8:30 – 9:30) — Member 1
[Screen: Back to architecture diagram or design document]
"Two design decisions I want to call out explicitly, because Marcus Rivera — our VP of Engineering stakeholder — pushed on both of these.
On storage: we chose RDS over DynamoDB because every business query requires GROUP BY aggregations. DynamoDB has no native GROUP BY support. To answer 'revenue by category' in DynamoDB, you'd need to scan the entire table or maintain a separate pre-aggregated table — adding complexity without benefit. RDS was the right tool for this access pattern.
On data quality: the framing Marcus challenged us on was — why drop some records but fix others? Our answer is that the decision depends on whether the core business data is intact. A missing payment method is a gap in one field, but the order ID, product, quantity, and price are all valid. We keep the order. A missing customer ID means we can't attribute the sale to anyone — the record has no analytical value. We drop it. Every decision is logged so the operations team knows exactly what was lost and why.
The broader point Marcus made — and he was right — is that data quality decisions have revenue implications. The records we drop represent real orders that won't appear in any revenue report. The operations team needs that audit trail to understand where POS data quality is degrading by store."

CLOSE (9:30 – 10:00) — Member 1
"To summarize: we built a working data lake on AWS that ingests 500 messy CSV files and one JSON file, applies six data quality rules, writes clean data to both S3 and RDS, and logs every quality issue. We analyzed the event-driven concurrency failure in detail and built a batch processor that solves it by design — one connection, one loop, no race conditions.
Thanks for watching. All code, screenshots, the submission JSON, and the stakeholder transcript are included with this submission."

SCREEN RECORDING CHECKLIST
Have these open before you hit record:

 S3 console — raw bucket with raw/ prefix expanded
 S3 console — processed bucket with processed/ prefix expanded
 Lambda console — batch processor function page
 CloudShell tab — already connected to RDS (run the mysql connect command before recording)
 Architecture diagram (from design document or this conversation)

Pre-run in CloudShell before recording the demo section so you know the results:
sqlUSE freshcart;
SELECT COUNT(*) AS total_clean_orders FROM clean_orders;
SELECT category, COUNT(*) AS order_count, ROUND(SUM(total_amount),2) AS total_revenue
  FROM clean_orders GROUP BY category ORDER BY total_revenue DESC;
SELECT shipping_city, COUNT(*) AS order_count FROM clean_orders
  GROUP BY shipping_city ORDER BY order_count DESC LIMIT 5;
SELECT payment_method, COUNT(*) AS usage_count FROM clean_orders
  GROUP BY payment_method ORDER BY usage_count DESC;
SELECT issue_type, COUNT(*) AS issue_count FROM data_quality_log
  GROUP BY issue_type ORDER BY issue_count DESC;
SELECT order_date, COUNT(*) AS daily_orders, ROUND(SUM(total_amount),2) AS daily_revenue
  FROM clean_orders GROUP BY order_date ORDER BY order_date;
SELECT COUNT(*) FROM customer_feedback;
SELECT * FROM customer_feedback LIMIT 5;
If CloudShell disconnects mid-recording, reconnect with:
bashmysql -h mp1-558-teame5-rds.csxdborekups.us-east-1.rds.amazonaws.com -u admin123 -p
