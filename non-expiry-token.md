## Non Expiry token

```

export USR=<admin>

export PASSWORD=<password>

export CP4D_URL=https://<cpd-url>

TOKEN=`curl -u ${USR}:${PASSWORD} ${CP4D_URL}/v1/preauth/validateAuth -k | jq '.accessToken' --raw-output`

curl -k -X POST ${CP4D_URL}/api/v1/usermgmt/v1/usermgmt/getTimedToken -H "Authorization: Bearer $TOKEN" -H "lifetime: 0"

```
