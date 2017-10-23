# Analyse _all_ package.json's in Github

Matthias Luebken, 05. July 2016

## TL;DR
I've just analyzed over a million package.json's on Github and counted all dependencies. The top 10000 packages used are https://docs.google.com/spreadsheets/d/1SL25G6w3aLDcSzgsCOYmyJVlc6Jdmu0SuE070J56UNo/edit#gid=2081634668

## Context

Last week Google and Github announced the possibilty to query all data with Googles Bigquery. This is particular exciting for tools like Bayesian as it allows to analayse Open Source code without downloading anything. I've spend a couple of hours to create a small spike. I think this has great potential for further qualifying our upstream popularity data.

## Links:

* Announcement: https://github.com/blog/2201-making-open-source-data-more-available
* Links to articles / tutorials: https://medium.com/@hoffa/github-on-bigquery-analyze-all-the-code-b3576fd2b150#.uvl0q2buv

# Spike

Note: The following is just a rough POC spike and has propably lots of errors.  

## Create a table with the contents of all package.json

This query uses `sample_contents` as an input. Switch the comments in the appropriate lines to get all the data. Caution: That will result in a big query (> 1 TB) which will result in costs.

```
SELECT
  * FROM
  #[bigquery-public-data:github_repos.sample_contents]
  [bigquery-public-data:github_repos.contents] AS CONTENTS
JOIN (
  SELECT
    id FROM
    #[bigquery-public-data:github_repos.sample_files]
    [bigquery-public-data:github_repos.files]
  WHERE
    path LIKE '%package.json'
  GROUP BY
    id) AS FIELDS
ON
  CONTENTS.id = FIELDS.id
```

Save the contents in a table. e.g. data set: mdl_github_repos, table: sample_package_jsons

For selecting the real data I had to `allow large results` for this project. When I ran this it took 174.2s elapsed and processed 1.74 TB processed. I got 1346428 package.jsons!

## Analyse package.jsons

With the table above we can now walk through package.jsons contents get the dependencies and count the packages. In this spike I've omitted versions but this can be easily added. 

```
SELECT
  REGEXP_EXTRACT(name_version, r'\"([\w-@]*)') AS dependency_name,
  COUNT(dependency_name) AS count_name
FROM (
  SELECT
    SPLIT( RTRIM( LTRIM( JSON_EXTRACT(CONTENTS_content, '$.dependencies'), '{' ), '}' ) ) AS name_version
  FROM
    [mdl-bigquery:mdl_github_repos.sample_package_jsons] )
    #[mdl-bigquery:mdl_github_repos.package_jsons] )
GROUP BY
  dependency_name
ORDER BY
  count_name DESC
```

Query complete (7.6s elapsed, 3.23 GB processed)

The top 10:
```
Row	dependency_name	count_name	 
1	lodash	107781	 
2	express	66433	 
3	debug	47323	 
4	async	39942	 
5	body-parser	35572	 
6	inherits	35312	 
7	request	31159	 
8	underscore	29327	 
9	chalk	25825	 
10	mkdirp	25797	
```

The top 10000: https://docs.google.com/spreadsheets/d/1SL25G6w3aLDcSzgsCOYmyJVlc6Jdmu0SuE070J56UNo/edit#gid=2081634668

## Next steps

As written above this is just a rough spike. As soon we have priority on upstream popularity we can evaluate theses scripts inculde versions and ingest the data into Bayesian. I would particular find it interesting to further qualify the repositories and not count all package.jsons as equal.

