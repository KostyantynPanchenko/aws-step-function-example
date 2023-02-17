# AWS Step Function Example

This is a modified example of [Getting started with AWS Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/getting-started-with-sfn.html).

---

## Step Function Definition
```json
{
  "Comment": "A description of my state machine",
  "StartAt": "Get credit limit",
  "States": {
    "Get credit limit": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "Payload.$": "$",
        "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:RandomNumberForCredit:$LATEST"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 6,
          "BackoffRate": 2
        }
      ],
      "Next": "Credit applied >= 5000?",
      "ResultSelector": {
        "withdrawAmount.$": "$.Payload"
      },
      "ResultPath": "$.GetCreditLimitResult"
    },
    "Credit applied >= 5000?": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.GetCreditLimitResult.withdrawAmount",
          "NumericLessThan": 5000,
          "Next": "Auto-approve limit"
        },
        {
          "Variable": "$.GetCreditLimitResult.withdrawAmount",
          "NumericGreaterThanEquals": 5000,
          "Next": "Credit limit approved"
        }
      ],
      "Default": "Wait for human approval"
    },
    "Wait for human approval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish.waitForTaskToken",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:StepFunctionTaskTokenTopic",
        "Message": {
          "TaskToken.$": "$$.Task.Token"
        }
      },
      "Next": "Credit limit approved"
    },
    "Credit limit approved": {
      "Type": "Pass",
      "Next": "Verify applicant's identity and address",
      "ResultPath": null
    },
    "Verify applicant's identity and address": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "Verify identity",
          "States": {
            "Verify identity": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "OutputPath": "$.Payload",
              "Parameters": {
                "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:check-identity:$LATEST",
                "Payload.$": "$"
              },
              "Retry": [
                {
                  "ErrorEquals": [
                    "Lambda.ServiceException",
                    "Lambda.AWSLambdaException",
                    "Lambda.SdkClientException",
                    "Lambda.TooManyRequestsException"
                  ],
                  "IntervalSeconds": 2,
                  "MaxAttempts": 6,
                  "BackoffRate": 2
                }
              ],
              "End": true,
              "InputPath": "$.data.identity"
            }
          }
        },
        {
          "StartAt": "Verify address",
          "States": {
            "Verify address": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "OutputPath": "$.Payload",
              "Parameters": {
                "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:check-address:$LATEST",
                "Payload.$": "$"
              },
              "Retry": [
                {
                  "ErrorEquals": [
                    "Lambda.ServiceException",
                    "Lambda.AWSLambdaException",
                    "Lambda.SdkClientException",
                    "Lambda.TooManyRequestsException"
                  ],
                  "IntervalSeconds": 2,
                  "MaxAttempts": 6,
                  "BackoffRate": 2
                }
              ],
              "End": true,
              "InputPath": "$.data.address"
            }
          }
        }
      ],
      "Next": "Get list of credit bureaus"
    },
    "Get list of credit bureaus": {
      "Type": "Task",
      "Parameters": {
        "TableName": "GetCreditBureau"
      },
      "Resource": "arn:aws:states:::aws-sdk:dynamodb:scan",
      "Next": "Get scores from all credit bureaus"
    },
    "Get scores from all credit bureaus": {
      "Type": "Map",
      "ItemProcessor": {
        "ProcessorConfig": {
          "Mode": "INLINE"
        },
        "StartAt": "Get all scores",
        "States": {
          "Get all scores": {
            "Type": "Task",
            "Resource": "arn:aws:states:::lambda:invoke",
            "OutputPath": "$.Payload",
            "Parameters": {
              "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:get-credit-score:$LATEST"
            },
            "Retry": [
              {
                "ErrorEquals": [
                  "Lambda.ServiceException",
                  "Lambda.AWSLambdaException",
                  "Lambda.SdkClientException",
                  "Lambda.TooManyRequestsException"
                ],
                "IntervalSeconds": 2,
                "MaxAttempts": 6,
                "BackoffRate": 2
              }
            ],
            "End": true
          }
        }
      },
      "End": true,
      "ItemsPath": "$.Items"
    },
    "Auto-approve limit": {
      "Type": "Pass",
      "Next": "Verify applicant's identity and address",
      "ResultPath": null
    }
  }
}
```

---

## GetCreditLimit lambda

The same as in example.
```javascript
export const handler = async function(event, context) {
    const credLimit = Math.floor(Math.random() * 10000);
    return (credLimit);
    
};
```

---

## Check identity lambda

```javascript
export const handler = async event => {
  const {
    ssn,
    email
  } = event;
  console.log(`SSN: ${ssn} and email: ${email}`);

  const approved = [ssn, email].every(i => i?.trim().length > 0);

  return {
    statusCode: 200,
    body: JSON.stringify({
      approved,
      message: `Identity validation ${approved ? 'passed' : 'failed'}`
    })
  }
};
```

---

## Check address lambda

```javascript
export const handler = async event => {
  const {
    street,
    city,
    state,
    zip
  } = event;
  console.log(`Address information: ${street}, ${city}, ${state} - ${zip}`);

  const approved = [street, city, state, zip].every(i => i?.trim().length > 0);

  return {
    statusCode: 200,
    body: JSON.stringify({
      approved,
      message: `Address validation ${ approved ? 'passed' : 'failed'}`
    })
  }
};
```

---

## Payload sample

```json
{
  "data": {
    "firstname": "Michael",
    "lastname": "Jordan",
    "identity": {
      "email": "mj23@jordan.com",
      "ssn": "123-45-6789"
    },
    "address": {
      "street": "123 Main St",
      "city": "New York",
      "state": "NY",
      "zip": "80600"
    }
  }
}
```

---

## Useful links

* [Getting started with AWS Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/getting-started-with-sfn.html)
* [Data flow simulator](https://us-east-1.console.aws.amazon.com/states/home?region=us-east-1#/simulator)
