---
date: 2020-02-09
title: "Migrating to Hugo from Jykell"
---


The other day, I was working on a project to consume video data in Kafka, the original service was written in python. Here is an example:

{{< highlight python >}}
from kafka import KafkaProducer
from kafka.errors import KafkaError

producer = KafkaProducer(bootstrap_servers='localhost:9092')

topic = 'my-topic'

video = cv2.VideoCapture("cool-video.mp4")

while video.isOpened():
    success, frame = video.read()
    if not success:
        break

    data = cv2.imencode('.jpeg', frame)[1].tobytes()

    future = producer.send(topic, data)
    try:
        future.get(timeout=10)
    except KafkaError as e:
        print(e)
        break
{{< /highlight >}}
