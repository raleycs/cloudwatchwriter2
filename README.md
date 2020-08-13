# zerolog2cloudwatch
Package to enable sending logs from zerolog to AWS CloudWatch.

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

### Standard use case
If you want zerolog to send all logs to CloudWatch then do the following:
```
import (
	"fmt"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/mec07/zerolog2cloudwatch"
	"github.com/rs/zerolog/log"
)

const (
    region = "eu-west-2"
    logGroupName = "log-group-name"
    logStreamName = "log-stream-name"
)

func setupZerolog(accessKeyID, secretKey string) error {
	sess, err := session.NewSession(&aws.Config{
		Region:      aws.String(region),
		Credentials: credentials.NewStaticCredentials(accessKeyID, secretKey, ""),
	})
	if err != nil {
		return log.Logger, fmt.Errorf("session.NewSession: %w", err)
	}

	cloudwatchWriter, err := zerolog2cloudwatch.NewWriter(sess, logGroupName, logStreamName)
	if err != nil {
		return log.Logger, fmt.Errorf("zerolog2cloudwatch.NewWriter: %w", err)
	}

	log.Logger = log.Output(cloudwatchWriter)
}
```
If you prefer to use AWS IAM credentials that are saved in the usual location on your computer then you don't have to specify the credentials, e.g.:
```
sess, err := session.NewSession(&aws.Config{
    Region:      aws.String(region),
})
```
For more details, see: https://docs.aws.amazon.com/sdk-for-go/api/aws/session/.
See the example directory for a working example.

### Write to CloudWatch and the console
What I personally prefer is to write to both CloudWatch and the console, e.g.
```
cloudwatchWriter, err := zerolog2cloudwatch.NewWriter(sess, logGroupName, logStreamName)
if err != nil {
    return fmt.Errorf("zerolog2cloudwatch.NewWriter: %w", err)
}
consoleWriter := zerolog.ConsoleWriter{Out: os.Stdout}
log.Logger = log.Output(zerolog.MultiLevelWriter(consoleWriter, cloudwatchWriter))
```

### Create a new zerolog Logger
Of course, you can create a new `zerolog.Logger` using this too:
```
cloudwatchWriter, err := zerolog2cloudwatch.NewWriter(sess, logGroupName, logStreamName)
if err != nil {
    return fmt.Errorf("zerolog2cloudwatch.NewWriter: %w", err)
}
logger := zerolog.New(cloudwatchWriter).With().Timestamp().Logger()
```
and of course you can create a new `zerolog.Logger` which can write to both CloudWatch and the console:
```
cloudwatchWriter, err := zerolog2cloudwatch.NewWriter(sess, logGroupName, logStreamName)
if err != nil {
    return fmt.Errorf("zerolog2cloudwatch.NewWriter: %w", err)
}
consoleWriter := zerolog.ConsoleWriter{Out: os.Stdout}
logger := zerolog.New(zerolog.MultiLevelWriter(consoleWriter, cloudwatchWriter)).With().Timestamp().Logger()
```


## Acknowledgements
Much thanks has to go to the creator of `zerolog` (https://github.com/rs/zerolog), for creating such a good logger.
Thanks must go to the writer of `logrus-cloudwatchlogs` (https://github.com/kdar/logrus-cloudwatchlogs) as I found it a helpful resource for interfacing with `cloudwatchlogs`.
Thanks also goes to the writer of this: https://gist.github.com/asdine/f821abe6189a04250ae61b77a3048bd9, which I also found helpful for extracting logs from `zerolog`.