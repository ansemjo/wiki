# Gitlab API

[Official Documentation](https://docs.gitlab.com/ee/api/README.html) is available with all the v4
API routes.

## Bash Alias

A useful bash alias for `httpie` to interact with the GitLab API:

```bash
gitlab() {
  meth=${1:?http method};
  api=${2:?api path};
  shift 2;
  http --check-status \
    "$meth" "https://git.rz.semjonov.de/api/v4/$api" \
    private-token:"$TOKEN" \
    "$@";
}
```

Then export your [personal access token](https://git.rz.semjonov.de/profile/personal_access_tokens)
to env:

```
read TOKEN && export TOKEN
```

## Usage

Chained to `jq`, the usage becomes:

```bash
$ gitlab GET projects | jq 'map(.name)'
[
  "deploy",
  "bookstack",
  "preseedinjector",
  "frontend",
  "sbupdate",
  "..."
]
```

```bash
$ gitlab PUT projects/11 wiki_enabled=false
HTTP/1.1 200 OK
Cache-Control: max-age=0, private, must-revalidate
Connection: keep-alive
Content-Length: 2022
Content-Type: application/json
Date: Fri, 27 Jul 2018 13:41:41 GMT
...
```

### Examples

#### Get the Wiki Status

A stupid loop to get the `wiki_enabled` status of projects:

```bash
for i in {1..116}; do
  project=$(gitlab GET projects/$i 2>/dev/null) \
  && wiki=$(jq .wiki_enabled <<<"$project") \
  && path=$(jq .path_with_namespace <<<"$project") \
  && echo "$i $path: $wiki";
done
```
