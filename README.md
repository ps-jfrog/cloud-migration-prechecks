#  Collection of scripts that automate the migration pre-checks required before [migration to JFrog Cloud](https://docs.jfrog-applications.jfrog.io/jfrog-applications/jfrog-cli/cli-for-jfrog-cloud-transfer)


## [Transferring files larger than 25GB](https://docs.jfrog-applications.jfrog.io/jfrog-applications/jfrog-cli/cli-for-jfrog-cloud-transfer#running-the-transfer-process-exceptional-cases)

> If the size value you received is larger than 25GB, please avoid initiating the files transfer before contacting JFrog Support, to check whether this size limit can be increased for you. You can contact Support by sending an email to support@jfrog.com

```bash
# Return largest artifacts

curl --location "$JF_URL/artifactory/api/search/aql" \
--header "Content-Type: text/plain" \
--header "Authorization: Bearer $BEARER_TOKEN" \
--data 'items.find({"name" : {"$match":"*"}}).include("size","name","repo").sort({"$desc" : ["size"]}).limit(1)' \
| jq '.results[] |= (.size = (.size / 1073741824))'
```

## Docker repostirories naming conventions

> Docker repositories with names that include dots or underscores aren't allowed in JFrog Cloud.

```bash
JF_URL="https://psemea.jfrog.io" \
BEARER_TOKEN="eyJ2Z...."

curl --location "$JF_URL/artifactory/api/repositories?packageType=docker" \
--header "Authorization: Bearer $BEARER_TOKEN" | jq '.[] | select(.key | contains(".") or contains("_"))'
```

## Artifacts with long properties string value

> Artifact properties with a value longer than 2.4K characters are not supported in JFrog Cloud. Such properties are generally seen in Conan artifacts. The artifacts will be transferred without the properties in this case. A report with these artifacts will become available to you at the end of the transfer.

```bash
curl --location "$JF_URL/metadata/api/v1/query" \
--header "Content-Type: text/plain" \
--header "Authorization: Bearer $BEARER_TOKEN" \
--data '{ "query": "{ packages(filter: {name: \"*\"}, first: 3, orderBy: {field: NAME, direction: DESC}) { edges { node { name packageType properties { name value } } } } }" }' \
| jq '.data.packages.edges[] | select(.node.properties[]?.value | length > 24000) | .node'
```

## User plugins are not supported on JFrog Cloud.

```bash
curl --location "$JF_URL/artifactory/api/plugins" \
--header "Authorization: Bearer $BEARER_TOKEN"
```



