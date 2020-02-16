---
date: 2020-02-09
title: "Migrating to Hugo from Jykell"
---

Nearly six months ago, I found myself working on a project to consume video data in [Apache Kafka](https://kafka.apache.org/). The original service was written in [Python](https://www.python.org/).

Here is an example Kafka producer that sent video data to a topic:

{{< highlight python >}}
from kafka import KafkaProducer
from kafka.errors import KafkaError

producer = KafkaProducer(bootstrap_servers='localhost:9092')

topic = 'my-topic'

video_path = "cool-video.mp4"

chunk_size = 5 * 1024 * 1024  # 5mb

with io.open(video_path, 'rb') as video_file:

    while True:
        data = video_file.read(chunk_size)
        if not data:
            break
        future = producer.send(topic, data)
        try:
            future.get(timeout=10)
        except KafkaError as e:
            print(e)
            break

{{< /highlight >}}

In this setup, there was a parallel consumer process that sent the produced data to [Google Video Intelligence API](https://cloud.google.com/video-intelligence) for enrichment and put the results on a new topic, which would later be joined with the raw data:

{{< highlight python >}}
from kafka import KafkaConsumer, KafkaProducer
from google.cloud import videointelligence_v1p3beta1 as videointelligence

topic = 'my-topic'
enriched_topic = 'my-topic-enriched'

consumer = KafkaConsumer(topic, bootstrap_servers='localhost:9092')
producer = KafkaProducer(bootstrap_servers='localhost:9092')

client = videointelligence. StreamingVideoIntelligenceServiceClient()

config = videointelligence.types. StreamingVideoConfig(

    feature=(videointelligence.enums.StreamingFeature.
                STREAMING_SHOT_CHANGE_DETECTION))

config_request = videointelligence.types. StreamingAnnotateVideoRequest(

    video_config=config)

def stream_generator():

    yield config_request
    for message in consumer:
        yield videointelligence.types.StreamingAnnotateVideoRequest(
            input_content=message.value)

requests = stream_generator()

responses = client.streaming_annotate_video(requests, timeout=300)

for response in responses:

    if response.error.message:
        break

    future = producer.send(enriched_topic, response)
    try:
        future.get(timeout=10)
    except KafkaError as e:
        print(e)
        break

{{< /highlight >}}

If you are a true python hero, you might see this code example and instantly think, "Why not use `aiokafka` package to provide `async` features, which should optimize the performance?" Then you'd have a Machine Learning pipeline that many companies dream of implementing.

I've been a Python cheerleader since version 2.4. It's not that I don't like Java or C++; I simply appreciate how Python got the scientific community excited about solving complicated problems in new ways. The idea that you don't have to be an engineer who understands [SOLID principles](https://en.wikipedia.org/wiki/SOLID) so you can write a basic program that consumes and analyzes image data from the [Hubble Telescope](https://hla.stsci.edu/) is worth celebrating.

Turning back to the original point of the Kafka video enrichment example above- I wanted to increase the `chuck_size` from `5mb` to `10mb` , without taking a hit on performance. To do that, I began to explore [Golang](https://golang.org/), which spoiler alert, certainly took me down a rabbit hole.

A few years back, like all engineers, I installed Golang (Go), added `GOPATH` to my `.zshrc` and ran a couple `go run .` and `go build .` on simple programs like this:

{{< highlight go >}}

package main
import "fmt"
func main() {

    fmt.Println("hello world")

}
{{< /highlight >}}

At that point I was convinced that Go is __neat__, and figured I should explore more sometime. Every now and then, I looked at `terraform` or `docker` internals, which are all written in Go, so I can understand how specific things worked. But I didn't really start my Go journey until this video processing project.

Rewriting the pipeline in Go was not that difficult. You can find the full example on [GitHub](https://github.com/zeyaddeeb/ml-video-pipeline). The performance was not that much better than Python. The main bottleneck is networking (making the API call to Google for enrichment) which is a solely language agnostic problem.

Running these processes in a distributed fashion Kuberenets (also built in Go) could also be problematic since it's easy to hit the rate limits for the Video Intelligence API. The other option is to use [aistreamer](https://github.com/google/aistreamer), which requires a different blog post all together.

While exploring another language didn't get me different results, it was still worth the effort. I know that Python is still most relevant when it comes to Machine Learning and Data Science, but Go became my second choice when developing production Machine Learning applications.

I wasn't really updating the blog as frequently as I would like. So I decided to start a new page with Hugo (which is written in Go) and update the blog more often as I learn more Go. Most of my code will still be in Python, but there will be some Go nuggets every now and then.
