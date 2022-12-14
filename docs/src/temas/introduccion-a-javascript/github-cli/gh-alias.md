---
sidebar: auto
next: gh-api.md
prev: gh.md
---

# Introduction to `gh alias`

## Manual

See <https://cli.github.com//manual/gh_alias_set>

## gh alias set

```
➜ gh alias set <alias> <expansion> [flags]
```

Declare a word as a command alias that will expand to the specified command(s).

The expansion may specify additional arguments and flags. If the expansion
includes positional placeholders such as `$1`, `$2`, etc., any extra arguments
that follow the invocation of an alias will be inserted appropriately.


If `--shell` is specified, the alias will be run through a shell interpreter (sh). 

* This allows you
to compose commands with `|` or redirect with `>`. 
* Note that extra arguments following the alias
will not be automatically passed to the expanded expression. 
* To have a shell alias receive
arguments, you must explicitly accept them using `$1`, `$2`, etc., or `$@` to accept all of them.

Quotes must always be used when defining a command as in the examples.

## Simple Example

La opción `--paginate` nos permite obtener todos los items. La opción `--jq` nos permite
filtrar los items usando [jq](https://stedolan.github.io/jq/).

```
➜ gh alias set org-list-names "api --paginate /user/memberships/orgs --jq '.[].organization.login'"
- Adding alias for org-list-names: api --paginate /user/memberships/orgs --jq '.[].organization.login'
✓ Added alias.
➜  gh org-list-names | grep 23
ULL-ESIT-DMSI-2223
ULL-ESIT-PL-2223
ULL-MII-SYTWS-2223
ULL-MFP-AET-2223
ULL-ESIT-LPP-2223
```

## Example search for members of an organization

Este request nos da un JSON con información sobre los miembros de una org:

```
➜ gh alias set org-ms 'api --paginate "/orgs/$1/members"'
- Adding alias for org-ms: api --paginate "/orgs/$1/members"
✓ Added alias.
```
  
Que lo podemos usar así:

```
➜ gh org-ms ULL-MII-SYTWS-2223 --jq '.[].login'
algorithms-ull
alu0101225562
alu0101229942
casiano
claudio4
crguezl
JobabaEV
SantiagoVV
```

## Example: Search for repos inside an organization

Let us search for repos inside our organization using GitHub API v3:

```
➜ gh api '/search/repositories?q=iaas+org:ULL-MII-SYTWS-2021+in:name'
```

![](/images/gh-api-search-for-repos.png) 

Here is [the JSON with the full output](gh-get-labs-output.json).

* See the [SEARCH](https://docs.github.com/en/free-pro-team@latest/rest/reference/search)
section of the REST API GitHub docs to know more about the API.
* See section [Search Repositories](https://docs.github.com/en/free-pro-team@latest/rest/reference/search#search-repositories) for more info on how to search for repos

Now we can use `gh alias set` to make an alias `get-lab` to get the repos:

```
➜ gh alias set get-labs 'api /search/repositories?q=$2+org:$1+in:name'
- Adding alias for get-labs: api /search/repositories?q=$2+org:$1+in:name
✓ Added alias.
➜  gh alias list
co:        pr checkout
get-labs:  api /search/repositories?q=$2+org:$1+in:name
```

And now we can use it:

```
➜ gh get-labs ULL-MII-SYTWS-2021 iaas
```

Next  we can pipe the output to [jq](jq) to get the names of the repos and the date of the last push:

```
➜ gh get-labs ULL-MII-SYTWS-2021 iaas | jq '.items[] | .name, .pushed_at'
"p01-t1-iaas-juanchojbarroso"
"2020-10-21T15:58:32Z"
"p01-t1-iaas-alu0101040882"
"2020-10-17T16:53:39Z"
"p01-t1-iaas-fcohdezc"
"2020-10-06T17:51:52Z"
"p01-t1-iaas-crguezl"
"2020-10-19T13:50:13Z"
"p01-t1-iaas-alu0100886870"
"2020-10-21T17:05:08Z"
"p01-t1-iaas-lardabi"
"2020-10-06T18:01:16Z"
```
We can improve it by writing a script:

```
➜  cat ~/bin/repos
```

```bash
#!/bin/bash

ORG=ULL-MII-SYTWS-2021
ASSIGNMENT=iaas
if [[ $# -gt 0 ]] ; then
  ASSIGNMENT=$1
fi
if [[ $# -gt 1 ]] ; then
    ORG=$2
fi
# echo $ASSIGNMENT $ORG
gh api --paginate /search/repositories?q=$ASSIGNMENT+org:$ORG+in:name |
                          jq '.items[] | .name, .pushed_at'           |
                          sed 'N;s/\n/ => /'
```

Let us make an alias for `gh`:

```
➜ gh alias set --shell get-repos 'repos $1 $2'
- Adding alias for get-repos: repos $1 $2
✓ Changed alias get-repos from !repos to !repos $1 $2
```

Watch the use of single quotes.

Let us use our new alias:

```
➜  apuntes git:(curso2021) gh get-repos TFA ULL-ESIT-PL-1920
"tfa-module-miguel-tfa" => "2020-09-04T09:40:57Z"
"tfa-daniel-tfa" => "2020-06-02T14:00:30Z"
"tfa-manuel-jorge-tfa" => "2020-09-13T21:40:24Z"
"tfa-basilio-tfa" => "2020-07-14T06:49:29Z"
"tfa-alien-tfa" => "2020-09-05T07:35:52Z"
"tfa-miguel-angel-tfa" => "2020-09-15T13:19:47Z"
"tfa-esther-sergio-tfa" => "2020-07-10T08:53:04Z"
...
```
