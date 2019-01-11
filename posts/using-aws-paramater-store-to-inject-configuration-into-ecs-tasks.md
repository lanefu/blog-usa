---
authors:
- Lane Jennison
tags:
- Paramater Store
- AWS Systems Manager
- 12-factor
- spring boot
- DevOps
- Fargate
date: 2019-01-06T12:21:50.000Z
title: "Using AWS Paramter store for 12-factor style ECS container configuration"
image: 
---
### Database connection information stored in AWS Systems Manger Parameter Store ###
Since we were using Spring Boot, we had a lot of configuration methods to choose from.   Due to the small scale of the application, we opted for 12-factor style config by setting environment variables for the JDBC path and credentials.   AWS Systems Manager Parameter Store was a simple solution that could easily be extended for more configurations.  We created Secure Strings for the database configuration where the keys were spring environment variables prefixed with the `/ourapp/db/` path.

To inject the configuration, the parameter store path prefix `/ourapp/db/` was defined as an environment variable in the service's task definition.   A function was added to the docker entrypoint shell script to extract the values from the parameter store and inject into the environment.   The only additional package we had to add to our container image was `jq`.


```
get_parameter_store() {

if [ -z "$PARAMETERS_PATH" ]; then
  echo "please set PARAMETERS_PATH in environment to parameter store path"
  exit 1
fi

echo "checking paramter store path ${PARAMTERS_PATH}"
PARAMETERS=$(aws ssm get-parameters-by-path --path ${PARAMETERS_PATH} --with-decryption)

for row in $(echo ${PARAMETERS} | jq -c '.Parameters' | jq -c '.[]'); do
    KEY=$(basename $(echo ${row} | jq -c '.Name'))
    VALUE=$(echo ${row} | jq -c '.Value')

    KEY=`echo ${KEY} | tr -d '"'`
    VALUE=`echo ${VALUE} | tr -d '"'`

    export ${KEY}=${VALUE}
done
}
```  

The aws cli tools in the container will automatically have access to any resources specified in the [Task Execution Role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html).   We added a policy for the task's role to access the parameter store.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ssm:GetParametersByPath",
                "ssm:GetParameters",
                "ssm:GetParameter"
            ],
            "Resource": "arn:aws:ssm:us-east-1:XXXXXXXXXXXXX:parameter/ourapp/*"
        }
    ]
}
```
