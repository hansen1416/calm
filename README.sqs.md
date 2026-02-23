`awslocal sqs create-queue --queue-name calm-email-queue.fifo --attributes "FifoQueue=true"`

`awslocal sqs list-queues`

{
    "QueueUrls": [
        "http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/calm-email-queue.fifo"
    ]
}


```
awslocal sqs get-queue-attributes \
    --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/calm-email-queue.fifo \
    --attribute-names All
```

awslocal sqs send-message \
    --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/calm-email-queue.fifo \
    --message-body "Hello World"


---------------------------

1. When user save email campaign, it will add a email campaign id into a SQS quque.

2. A lambda service will process this email campaign quque. And save a linear email sequece into DB. 

todo: how do we process the email campaign and its events?

3. 


SQS only store the message id.