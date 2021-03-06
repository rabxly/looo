# Step 1: get the commits/domain
In the BigQuery a PushEvent looks something like this:
| type      | public | payload                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | repo.id  | repo.name | repo.url                           | actor.id | actor.login | actor.url                       |
|-----------|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------|-----------|------------------------------------|----------|-------------|---------------------------------|
| PushEvent | true   | {"push_id":537278470,"size":1,"distinct_size":1,"ref":"refs/heads/master","head":"185ac10fcf1eb9c48e6f978cfdf5300f0ae541c8","before":"5a20b05d06032586f9b482e982ab709ce001709d","commits":[{"sha":"185ac10fcf1eb9c48e6f978cfdf5300f0ae541c8","author":{"email":"a454492e42fd9810e577ebee548c7e59bd883bca@live.com.au","name":"j-"},"message":"Rename core constructors","distinct":true,"url":"https://api.github.com/repos/j-/ok/commits/185ac10fcf1eb9c48e6f978cfdf5300f0ae541c8"}]} | 24749928 | j-/ok     | https://api.github.com/repos/j-/ok | 590992   | j-          | https://api.github.com/users/j- |

For us the payload column is important. That contains the emails. In our example it is a454492e42fd9810e577ebee548c7e59bd883bca@live.com.au. GitHub hashed the first part of the email, but this is not important for us now. We need only the domain from the email (live.com.au).

The query to count the commits/domain name looks like this:
```sql
## pre-2015 API
CREATE TEMP FUNCTION
  json2array(json STRING)
  RETURNS ARRAY<STRING>
  LANGUAGE js AS """
          return JSON.parse(json).map(x=>JSON.stringify(x));
        """;
WITH
  export_commit_data AS(
  SELECT
    event_id,
    event_type,
    created_at,
    repo_name,
    repo_id,
    repo_url,
    actor_name,
    push_id,
    commit_data
  FROM (
    SELECT
      *,
      ARRAY(
      SELECT
        JSON_EXTRACT(x,
          '$')
      FROM
        UNNEST(array_commits) x) AS commit_data
    FROM (
      SELECT
        id AS event_id,
        type AS event_type,
        created_at,
        repo.name AS repo_name,
        repo.id AS repo_id,
        repo.url AS repo_url,
        actor.login AS actor_name,
        JSON_EXTRACT_SCALAR(payload,
          "$.push_id") AS push_id,
        json2array(JSON_EXTRACT(payload,
            '$.shas')) AS array_commits
      FROM
        `githubarchive.day.20140201`
      WHERE
        type='PushEvent'
        AND JSON_EXTRACT(payload,
          "$.shas") IS NOT NULL )))
SELECT
  event_id,
  event_type,
  created_at,
  repo_id,
  repo_name,
  repo_url,
  REGEXP_REPLACE(repo_url, "https://github.com/|https://api.github.dev/repos/", "") AS repo_fullname,
  push_id,
  actor_name,
  commit_data,
  JSON_EXTRACT_SCALAR(commit_data,
    "$[0]") AS commit_sha,
  JSON_EXTRACT_SCALAR(commit_data,
    "$[1]") AS email,
  REGEXP_EXTRACT(JSON_EXTRACT_SCALAR(commit_data,
      "$[1]"), "@(.*)") AS email_domain,
  REGEXP_REPLACE(LOWER(TRIM(REGEXP_EXTRACT(JSON_EXTRACT_SCALAR(commit_data,
      "$[1]"), "@(.*)"))), "[^a-zA-Z0-9\\.]+", "") as email_domain_cleaned
FROM (
  SELECT
    event_id,
    event_type,
    created_at,
    repo_name,
    repo_id,
    repo_url,
    push_id,
    actor_name,
    commit_data,
  FROM
    export_commit_data
  CROSS JOIN
    UNNEST(export_commit_data.commit_data) AS commit_data )
 ```

 After 2015 the format of the payload changed a but and requires a slightly different query:

```sql
## post-2015 API
CREATE TEMP FUNCTION
  json2array(json STRING)
  RETURNS ARRAY<STRING>
  LANGUAGE js AS """
          return JSON.parse(json).map(x=>JSON.stringify(x));
        """;
WITH
  export_commit_data AS(
  SELECT
    event_id,
    event_type,
    created_at,
    repo_name,
    repo_id,
    repo_url,
    push_id,
    actor_name,
    commit_data
  FROM (
    SELECT
      *,
      ARRAY(
      SELECT
        JSON_EXTRACT(x,
          '$')
      FROM
        UNNEST(array_commits) x) AS commit_data
    FROM (
      SELECT
        id AS event_id,
        type AS event_type,
        created_at,
        repo.name AS repo_name,
        repo.id AS repo_id,
        repo.url AS repo_url,
        actor.login AS actor_name,
        JSON_EXTRACT_SCALAR(payload,
          "$.push_id") AS push_id,
        json2array(JSON_EXTRACT(payload,
            '$.commits')) array_commits
      FROM
        `githubarchive.day.20200112`
      WHERE
        type='PushEvent' )))
        
SELECT
  event_id,
  event_type,
  created_at,
  repo_id,
  repo_name,
  repo_url,
  repo_name AS repo_fullname,
  push_id,
  actor_name,
  commit_data,
  JSON_EXTRACT_SCALAR(commit_data,
    "$.sha") AS commit_sha,
  JSON_EXTRACT_SCALAR(commit_data,
    "$.author.email") AS email,
  REGEXP_EXTRACT(JSON_EXTRACT_SCALAR(commit_data,
      "$.author.email"), "@(.*)") AS email_domain,
  REGEXP_REPLACE(LOWER(TRIM(REGEXP_EXTRACT(JSON_EXTRACT_SCALAR(commit_data,
            "$.author.email"), "@(.*)"))), "[^a-zA-Z0-9\\.]+", "") AS email_domain_cleaned
FROM (
  SELECT
    event_id,
    event_type,
    created_at,
    repo_name,
    repo_id,
    repo_url,
    repo_name AS repo_fullname,
    push_id,
    actor_name,
    commit_data,
  FROM
    export_commit_data
  CROSS JOIN
    UNNEST(export_commit_data.commit_data) AS commit_data )
```

#Step 2
Count the number of commits per domain.

The final result looks like this:
| month      | email_domain             | domain_count |
|------------|--------------------------|--------------|
| 2015-01-01 | gmail.com                |       131357 |
| 2015-01-01 | users.noreply.github.com |         8802 |
| 2015-01-01 | python.org               |         5786 |
| 2015-01-01 | hotmail.com              |         4942 |
| 2015-01-01 | fhda.edu                 |         3888 |
| 2015-01-01 | yahoo.com                |         3216 |
| 2015-01-01 | etudes.org               |         2736 |
| 2015-01-01 | qq.com                   |         1955 |
| 2015-01-01 | sly.mn                   |         1908 |
| 2015-01-01 | foothill.edu             |         1848 |
# Step 2: exclude email providers
The heavy lifting was done by Big Query. We exported the result into one csv and after that we used the good old Jupyter Notebooks to clean up the data.

As you can see in the example result, not surprisingly, the first one is gmail.com. We have to remove the email providers from the list. The list of the most popular email provider’s domains is available here: https://gist.github.com/tbrianjones/5992856/. We used this list to clean up the result.
Combined with other blacklisted domains.

The full notebook can be found here https://gist.github.com/codersrankOrg/77073c25ce5f7e7e6d4d77750b46e899.
