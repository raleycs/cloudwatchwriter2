# cloudwatchwriter2

A robust Zerolog writer for AWS CloudWatch using Go SDK v2.

## Usage

This library assumes that you have IAM credentials to allow you to talk to AWS CloudWatch Logs.
The specific permissions that are required are:
- CreateLogGroup,
- CreateLogStream,
- DescribeLogStreams,
- PutLogEvents.

If these permissions aren't assigned to the user who's IAM credentials you're using then this package will not work.
There are two exceptions to that:
- if the log group already exists, then you don't need permission to CreateLogGroup;
- if the log stream already exists, then you don't need permission to CreateLogStream.

See [the example](example/example.go).

Make sure to close the writer to flush the queue, you can `defer` the `Close()` call in main.
The `Close()` function blocks until all the logs have been processed.

The library was tested with 1.18 as it uses the new `go.work` feature for the example, but it whould work with any Go version supported by Zerolog and AWS SDK v2.

### Write to CloudWatch and the console

What I personally prefer is to write to both CloudWatch and the console, e.g.

```
log.Logger := zerolog.New(zerolog.MultiLevelWriter(consoleWriter, cloudWatchWriter)).With().Timestamp().Logger()
```

### Changing the default settings

#### Batch interval

The logs are sent in batches because AWS has a maximum of 5 [PutLogEvents requests per second](https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutLogEvents.html) per log stream.
The default value of the batch period is 5 seconds, which means it will send the a batch of logs at least once every 5 seconds.
Batches of logs will be sent earlier if the size of the collected logs exceeds 1MB (another AWS restriction).
To change the batch frequency, you can set the time interval between batches to a smaller or larger value, e.g. 1 second:

```
err := cloudWatchWriter.SetBatchInterval(time.Second)
```

If you set it below 200 milliseconds it will return an error.

The batch interval is not guaranteed as two things can alter how often the batches get delivered:
- as soon as 1MB of logs or 10k logs have accumulated, they are sent (due to AWS restrictions on batch size);
- we have to send the batches in sequence (an AWS restriction) so a long running request to CloudWatch can delay the next batch.

## LICENSE

MIT

## Acknowledgements

The original library was written by mac07 (https://github.com/mec07/cloudwatchwriter). Credit to lzap (https://github.com/lzap) for changing the library to support SDKv2. The work is based on logrus implementation (https://github.com/kdar/logrus-cloudwatchlogs) and a gist (https://gist.github.com/asdine/f821abe6189a04250ae61b77a3048bd9)
