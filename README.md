# ClickHouse GitHub Events Example

## Purpose

This repository provides a comprehensive example of how to build a real-time data pipeline for ingesting, processing, and analyzing GitHub event data using ClickHouse and Kafka. The project includes:

*   A Docker Compose setup for easy deployment of ClickHouse and Kafka.
*   ClickHouse table schemas for storing GitHub events and pull request data.
*   Materialized views to automatically populate tables from Kafka topics.
*   Example queries for analyzing the data.

This project is intended for developers and data engineers who want to learn how to use ClickHouse and Kafka for real-time analytics on large-scale event data.

This project demonstrates how to use ClickHouse to store and query GitHub events. It includes a `docker-compose.yml`
file to set up a ClickHouse instance and a Kafka broker.

## Setup

1. Start the services using Docker Compose:

   ```bash
   docker-compose up -d
   ```

2. Connect to the ClickHouse instance using the ClickHouse client:

   ```bash
   docker compose exec clickhouse clickhouse-client --password changeme
   ```

## Schema

### Create table

``` clickhouse
create table github
(
    file_time           DateTime,
    event_type          Enum('CommitCommentEvent' = 1, 'CreateEvent' = 2, 'DeleteEvent' = 3, 'ForkEvent' = 4, 'GollumEvent' = 5, 'IssueCommentEvent' = 6, 'IssuesEvent' = 7, 'MemberEvent' = 8, 'PublicEvent' = 9, 'PullRequestEvent' = 10, 'PullRequestReviewCommentEvent' = 11, 'PushEvent' = 12, 'ReleaseEvent' = 13, 'SponsorshipEvent' = 14, 'WatchEvent' = 15, 'GistEvent' = 16, 'FollowEvent' = 17, 'DownloadEvent' = 18, 'PullRequestReviewEvent' = 19, 'ForkApplyEvent' = 20, 'Event' = 21, 'TeamAddEvent' = 22),
    actor_login         LowCardinality(String),
    repo_name           LowCardinality(String),
    created_at          DateTime,
    updated_at          DateTime,
    action              Enum('none' = 0, 'created' = 1, 'added' = 2, 'edited' = 3, 'deleted' = 4, 'opened' = 5, 'closed' = 6, 'reopened' = 7, 'assigned' = 8, 'unassigned' = 9, 'labeled' = 10, 'unlabeled' = 11, 'review_requested' = 12, 'review_request_removed' = 13, 'synchronize' = 14, 'started' = 15, 'published' = 16, 'update' = 17, 'create' = 18, 'fork' = 19, 'merged' = 20),
    comment_id          UInt64,
    path                String,
    ref                 LowCardinality(String),
    ref_type            Enum('none' = 0, 'branch' = 1, 'tag' = 2, 'repository' = 3, 'unknown' = 4),
    creator_user_login  LowCardinality(String),
    number              UInt32,
    title               String,
    labels              Array(LowCardinality(String)),
    state               Enum('none' = 0, 'open' = 1, 'closed' = 2),
    assignee            LowCardinality(String),
    assignees           Array(LowCardinality(String)),
    closed_at           DateTime,
    merged_at           DateTime,
    merge_commit_sha    String,
    requested_reviewers Array(LowCardinality(String)),
    merged_by           LowCardinality(String),
    review_comments     UInt32,
    member_login        LowCardinality(String)
) engine = MergeTree order by (event_type, repo_name, created_at);
```

### Create GitHub Queue table with Kafka engine

```clickhouse
    create table github_queue
    (
      file_time           DateTime,
      event_type          Enum('CommitCommentEvent' = 1, 'CreateEvent' = 2, 'DeleteEvent' = 3, 'ForkEvent' = 4, 'GollumEvent' = 5, 'IssueCommentEvent' = 6, 'IssuesEvent' = 7, 'MemberEvent' = 8, 'PublicEvent' = 9, 'PullRequestEvent' = 10, 'PullRequestReviewCommentEvent' = 11, 'PushEvent' = 12, 'ReleaseEvent' = 13, 'SponsorshipEvent' = 14, 'WatchEvent' = 15, 'GistEvent' = 16, 'FollowEvent' = 17, 'DownloadEvent' = 18, 'PullRequestReviewEvent' = 19, 'ForkApplyEvent' = 20, 'Event' = 21, 'TeamAddEvent' = 22),
      actor_login         LowCardinality(String),
      repo_name           LowCardinality(String),
      created_at          DateTime,
      updated_at          DateTime,
      action              Enum('none' = 0, 'created' = 1, 'added' = 2, 'edited' = 3, 'deleted' = 4, 'opened' = 5, 'closed' = 6, 'reopened' = 7, 'assigned' = 8, 'unassigned' = 9, 'labeled' = 10, 'unlabeled' = 11, 'review_requested' = 12, 'review_request_removed' = 13, 'synchronize' = 14, 'started' = 15, 'published' = 16, 'update' = 17, 'create' = 18, 'fork' = 19, 'merged' = 20),
      comment_id          UInt64,
      path                String,
      ref                 LowCardinality(String),
      ref_type            Enum('none' = 0, 'branch' = 1, 'tag' = 2, 'repository' = 3, 'unknown' = 4),
      creator_user_login  LowCardinality(String),
      number              UInt32,
      title               String,
      labels              Array(LowCardinality(String)),
      state               Enum('none' = 0, 'open' = 1, 'closed' = 2),
      assignee            LowCardinality(String),
      assignees           Array(LowCardinality(String)),
      closed_at           DateTime,
      merged_at           DateTime,
      merge_commit_sha    String,
      requested_reviewers Array(LowCardinality(String)),
      merged_by           LowCardinality(String),
      review_comments     UInt32,
      member_login        LowCardinality(String)
    )
  engine = Kafka('kafka:9092', 'github', 'clickhouse',
                 'JSONEachRow') settings kafka_thread_per_consumer = 0, kafka_num_consumers = 1;
```

### Create Materialized View

```clickhouse
    create materialized view github_mv to github as
select *
from github_queue;
```

# Produce the [data](https://datasets-documentation.s3.eu-west-3.amazonaws.com/kafka/github_all_columns.ndjson) to the topic

```bash
   cat github_all_columns.ndjson | kcat -P -b localhost:9092   -t github 
```

# Query the data

## Count of events by type

```clickhouse
select
  event_type,
  count() as event_count
from github
group by event_type
order by event_count desc;
```

## Top 10 active repositories

```clickhouse
select
  repo_name,
  count() as event_count
from github
group by repo_name
order by event_count desc
limit 10;
```

## Top 10 active users

```clickhouse
select
  actor_login,
  count() as event_count
from github
group by actor_login
order by event_count desc
limit 10;
```

### Create github_pr table

```clickhouse
create table github_pr
(
  pr_number      UInt32,
  pr_description String,
  opened_by      LowCardinality(String),
  pr_created_at  DateTime
)
  engine = MergeTree
    order by (pr_number);
```

### Create github_pr_queue table

```clickhouse
create table github_pr_queue
(
  pr_number      UInt32,
  pr_description String,
  opened_by      LowCardinality(String),
  pr_created_at  DateTime
)
  engine = Kafka('broker:29092', 'github_pr', 'clickhouse', 'JSONEachRow')
    settings kafka_thread_per_consumer = 0, kafka_num_consumers = 1;
```

### Create Materialized View for github_pr

```clickhouse
create materialized view github_pr_mv to github_pr as
select *
from github_pr_queue;
```

### Join github_mv and github_pr_mv

```clickhouse
select *
from github_mv g
       join github_pr_mv g2 on g.number = g2.pr_number
where g.created_at = '2019-09-23 11:25:54';
```
