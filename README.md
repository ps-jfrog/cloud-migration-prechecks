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

## APIs to assist in data verification after the transfer

Compare data storage
https://jfrog.com/help/r/jfrog-rest-apis/aql-execution
https://jfrog.com/help/r/jfrog-rest-apis/refresh-storage-summary-info 
https://jfrog.com/help/r/jfrog-rest-apis/get-storage-summary-info 
https://jfrog.com/help/r/jfrog-rest-apis/get-repositories-by-type-and-project 

Compare config
https://jfrog.com/help/r/jfrog-rest-apis/get-permission-targets
https://jfrog.com/help/r/jfrog-rest-apis/users
https://jfrog.com/help/r/jfrog-rest-apis/groups
https://jfrog.com/help/r/jfrog-rest-apis/permissions
https://jfrog.com/help/r/xray-rest-apis/get-watches
https://jfrog.com/help/r/xray-rest-apis/get-policies
https://jfrog.com/help/r/xray-rest-apis/get-ignore-rules
https://jfrog.com/help/r/jfrog-rest-apis/build-info

## Post Transfer Verification

**Why is the total file count on my source and target instances different after the files transfer finishes?**

It is expected to see sometimes significant differences between the files count on the source and target instances after the transfer ends. These differences can be caused by many reasons, and in most cases are not an indication of an issue. For example, Artifactory may include file cleanup policies that are triggered by the file deployment. This can cause some files to be cleaned up from the target repository after they are transferred.

**How can I validate that all files were transferred from the source to the target instance?**

There's actually no need to validate that all files were transferred at the end of the transfer process. JFrog CLI performs this validation for you while the process is running. It does that as follows.

- JFrog CLI traverses the repositories on the source instance and pushes all files to the target instance.

- If a file fails to reach the target instance or isn't deployed there successfully, the source instance logs this error with the file details.

- At the end of the transfer process, JFrog CLI provides you with a summary of all files that failed to be pushed.

- The failures are also logged inside the transfer directory under the JFrog CLI home directory. This directory is usually located at ~/.jfrog/transfer. Subsequent runs of the jf rt transfer-files command use this information for attempting another transfer of the files.

**Does JFrog CLI validate the integrity of files, after they are transferred to the target instance?**

Yes. The source Artifactory instance stores a checksum for every file it hosts. When files are transferred to the target instance, they are transferred with the checksums as HTTP headers. The target instance calculates the checksum for each file it receives and then compares it to the received checksum. If the checksums don't match, the target reports this to the source, which will attempt to transfer the file again at a later stage of the process.



