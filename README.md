# aws_open_console

```
#!/bin/bash
set -e

# urlencode用のfunction定義
urlencode() {
  local length="${#1}"
  for (( i = 0; i < length; i++ )); do
    local c="${1:i:1}"
    case $c in
      [a-zA-Z0-9.~_-]) printf "$c" ;;
      *) printf '%%%02X' "'$c"
    esac
  done
}

# Federatedユーザ定義(今回はsessionIssuertと同等=全権限を付与)
FED_USER=feduser # 任意の名前で動作するが、要求元ユーザを明示することが望ましい
POLICY_FILE=policy_all_allow.json
cat << EOF > ${POLICY_FILE}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
EOF

# FederationToken取得(有効期間を調節したい場合には、--durationオプションを変更)
FEDERATION_TOKEN=$(aws sts get-federation-token --name ${FED_USER} --policy file://./${POLICY_FILE} --duration-seconds 3600)

# セッション文字列生成
SESSION_ID=$(echo ${FEDERATION_TOKEN} | jq ".Credentials.AccessKeyId" | tr -d "\"")
SESSION_KEY=$(echo ${FEDERATION_TOKEN} | jq ".Credentials.SecretAccessKey" | tr -d "\"")
SESSION_TOKEN=$(echo ${FEDERATION_TOKEN} | jq ".Credentials.SessionToken" | tr -d "\"")

# 一時ファイル削除
rm ${POLICY_FILE}

# SigninToken取得
JSON_FORMED_SESSION=$(echo "{\"sessionId\":\"${SESSION_ID}\",\"sessionKey\":\"${SESSION_KEY}\",\"sessionToken\":\"${SESSION_TOKEN}\"}")
SIGNIN_URL="https://signin.aws.amazon.com/federation"
GET_SIGNIN_TOKEN_URL="${SIGNIN_URL}?Action=getSigninToken&SessionType=json&Session=$(urlencode ${JSON_FORMED_SESSION})"
SIGNIN_TOKEN=$(curl -s "${GET_SIGNIN_TOKEN_URL}" | jq ".SigninToken" | tr -d "\"")

# LOGIN URLの出力
ISSUER_URL="https://mysignin.internal.mycompany.com/"
CONSOLE_URL="https://console.aws.amazon.com/"
LOGIN_URL="${SIGNIN_URL}?Action=login&Issuer=$(urlencode ${ISSUER_URL})&Destination=$(urlencode ${CONSOLE_URL})&SigninToken=$(urlencode ${SIGNIN_TOKEN})" && echo ${LOGIN_URL}
 
open $LOGIN_URL
```
