# Splunk investigation queries

Set the lab time picker to **All time**. The searches use the lab's `scenario` index and sourcetypes. Replace `user@example.test` and `2001:db8::10` with the account and IP found in your results.

## 1. Group failed sign-ins by source IP

```spl
index=scenario sourcetype="azure:aad:signin" "status.errorCode"!=0
| stats count as failures dc(userPrincipalName) as accounts
    values(userPrincipalName) as targeted_accounts by ipAddress
| sort - failures
```

## 2. Rebuild the sign-in timeline

```spl
index=scenario sourcetype="azure:aad:signin" ipAddress="2001:db8::10"
| sort 0 + _time
| table _time userPrincipalName ipAddress appDisplayName location.city status.errorCode
```

Find the first code `0` after the failure sequence. Remove the table line and use **Events** when you need Splunk's exact displayed timestamp.

## 3. Review identity changes

```spl
index=scenario sourcetype="azure:aad:audit"
    (targetResources{}.userPrincipalName="user@example.test" OR initiatedBy.user.userPrincipalName="user@example.test")
| eval initiator=coalesce('initiatedBy.user.userPrincipalName', 'initiatedBy.app.displayName')
| sort 0 + _time
| table _time initiator activityDisplayName result targetResources{}.userPrincipalName
```

Open relevant raw events to review `targetResources.modifiedProperties`.

## 4. Review Microsoft 365 operations

```spl
index=scenario sourcetype="o365:management:activity" UserId="user@example.test"
| eval sourceIP=coalesce(ClientIP, ClientIPAddress)
| sort 0 + _time
| table _time Operation UserId sourceIP Workload ObjectId
```

Compare addresses across log sources. The inbox-rule record in this lab displayed the IPv6 address in brackets followed by a port.

## 5. Match the sent message to the accessed reply

```spl
index=scenario sourcetype="o365:management:activity" UserId="user@example.test"
    (Operation="Send" OR Operation="MailItemsAccessed")
| sort 0 + _time
```

Read `Item.Subject` on the send event. On the access event, expand each folder and its `FolderItems`. Use the reply subject and message identifiers to distinguish the reply from the original, and record the path from the folder entry that contains the reply.
