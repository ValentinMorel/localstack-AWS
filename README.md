# Deploy application on local mocked AWS : Localstack  
![AWS](https://miro.medium.com/max/2000/1*b_al7C5p26tbZG4sy-CWqw.png)

Requires : 
- Docker
- Docker-compose
- make
## Objective
The objective of this work aims to deploy a contenairized chatbot application that we can call on the cloud. 
It seems to need an entrypoint like a main where handler will process events and trigger actions.

## Setting up Localstack
### What is localstack ?

LocalStack provides an easy-to-use test/mocking framework for developing Cloud applications.
Currently, the focus is primarily on supporting the AWS cloud stack.

### Services available in the free version
LocalStack spins up the following core Cloud APIs on your local machine.

- ACM
- API Gateway
- CloudFormation
- CloudWatch
- CloudWatch Logs
- DynamoDB
- DynamoDB Streams
- EC2
- Elasticsearch Service
- EventBridge (CloudWatch Events)
- Firehose
- IAM
- Kinesis
- KMS
- Lambda
- Redshift
- Route53
- S3
- SecretsManager
- SES
- SNS
- SQS
- SSM
- StepFunctions
- STS

### Dockerfile 

``` Dockerfile
version: '2.1'
services:
  localstack:
    image: localstack/localstack-full
    container_name: localstack-sample
    ports:
      - "4566-4620:4566-4620"
      - "${PORT_WEB_UI-8080}:${PORT_WEB_UI-8080}"
    environment:
      - SERVICES=lambda
      - DEBUG=1
      - DEFAULT_REGION=us-east-2
      - DATA_DIR=/tmp/localstack/data
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "${TMPDIR:-/tmp/localstack}:/tmp/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

Run it with : 
```sh
docker-compose up -d
```

## A working sample

### The type of code we need as entrypoint
```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"

	"github.com/aws/aws-lambda-go/lambda"
)

type event struct {
	value1 string `json:"value2"`
	value2  string `json:"value2"`
}

func handleRequest(ctx context.Context, evt event) (string, error) {
	log.Printf("first value: %s\n", evt.value1)
	log.Printf("second value: %s\n", evt.value2)
	response := response{
		Name: fmt.Sprintf("%s %s", evt.value1, evt.value2),
	}
	byt, _ := json.Marshal(response)
	resp := string(byt)
	log.Println(resp)
	return resp, nil
}

func main() {
	lambda.Start(handleRequest)
}
```

The handler will process the data in the curl request.

### Sending to AWS lambda

Execute this command to zip the function and send to lambda : 

```sh
ENDPOINT_URL=http://localhost:4566
FUNCTION=sample-lambda
ROLE=role

go build -gcflags='all=-N -l' main.go

aws lambda create-function \
		--endpoint-url $(ENDPOINT_URL) \
		--runtime go1.x \
		--role $(ROLE) \
		--function-name $(FUNCTION) \
		--handler main \
		--zip-file fileb://function.zip
```

The runtime line refers to a docker image that contenairize a language environment to be able to run the program without caring about the credentials problem, environment settings, etc.
It gets rid of most of the problems at runtime.
The list of the available environment is in this link : [runtime environment for localstack lambda](https://github.com/lambci/docker-lambda)

### Invoking the function from AWS lambda

Execute then this command to call the function from AWS lambda : 
```sh
aws --endpoint-url http://localhost:4566 lambda invoke \
  --function-name sample-lambda \
  --payload '{"value1": "test", "value2": "test"}' \
  output
```

A file called "output" should be created. The content can be read with 'cat' command.
