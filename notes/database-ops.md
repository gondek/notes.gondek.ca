# SQL Databases: Operations and Upgrades

_Written: March 24, 2025._

## General Resources
- The PostgreSQL consultancy Dalibo has built a lovely [Query Plan Visualizer](https://github.com/dalibo/pev2). A hosted version is available at [explain.dalibo.com](https://explain.dalibo.com/). I use this frequently when analyzing query performance.
- While [Designing Data-Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/) by  Martin Kleppmann covers much more than just SQL, I do enjoy rereading chapters when I run into database design questions or performance issues.
- Markus Winand's [Use the Index Luke](https://use-the-index-luke.com/) is a fairly quick read with high payoff to understand query performance and index usage.


## AWS Aurora: Blue-Green Upgrades Runbook

At most of my workplaces, I have used MySQL or PostgreSQL hosted on AWS Aurora. Occasionally I had to upgrade the database engine to a new version, and I found working with [RDS blue/green deployments](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments.html) to be quite pleasant. 

... runbook coming soon ...