---
date: 2020-12-12
title: "TensorFlow Lite: Adventures in Scaling Production Models"
summary: "Scaling models in production is hard. Hopefully this post helps you get some of your research done before setting out on your own scaling journey.

TL;DR There's a good reason why everyone says to set Tensorflow thread counts to 1."
---

As a Data Scientist, I not only need to know how to develop and train models, I also need to deploy them to production to actually perform a their intended function, be it add face filter on your silly video (think: Snapchat), or a personal assistant which answers questions fo you (think: Siri).

For a long time, the tool I most regularly used for building deep learning models is TensorFlow. I occasionally got questions about why I don't use Pytorch instead, since it's newer and requires less effort. The answer is that I have actually used it, even since the time when it was recommended for "Research Use Only." Back then, it caused quite a commotion with dynamic graphs! That took some getting used to. Fo all intents and purposes, it'd been easier for me to stick with TensorFlow. As it happens, when TensorFlow 2 came out it, too, used dynamic graphs, and at that point I felt I just had to get on board. Opting to stick with TensorFlow has been a [Path of Least Resistance](https://en.wikipedia.org/wiki/Path_of_least_resistance) tactic, nothing more.

Anyhow, this is not a Tensorflow vs. Pytorch comparison. There are already way too many Medium articles which lay out all the differences. For now, my main interest is in introducing you to [Tensorflow Lite](https://www.tensorflow.org/lite).

From the authors:

> TensorFlow Lite is an open source deep learning framework for on-device inference. Works in 4 Steps:
>
> - Pick a model: Pick a new model or retrain an existing one.
> - Convert: Convert a TensorFlow model into a compressed flat buffer with the TensorFlow Lite Converter.
> - Deploy: Take the compressed .tflite file and load it into a mobile or embedded device.
> - Optimize: Quantize by converting 32-bit floats to more efficient 8-bit integers or run on GPU.

To summarize, you can take an existing `model.tf` file and convert it to `model.tflite` which gives you added benefits like reducing file size (using quantization) and making it compatible with mobile devices. It's currently true that not all models can be converted, since some deep learning graph operations are not yet supported. Still, it's been my experience that they typically the core Tensorflow development team adds new operations pretty quickly. Personally, all my model conversions needs can be met with `Tensorflow 2.4.0-rc1`)

As with everything, there are alternatives to TensorFlow Lite when attempting to deploy production models. For example, you could serve the model with [TFX](https://www.tensorflow.org/tfx) as a webserver. While this is good, it leaves you without the ability to do event-driven predictions (Kakfa enrichment, for example). At that point, there's still the potential that you'd run into all the common issues one tends to have with webservers, like failures due to networking issues, timeouts, high workloads, etc... Apart from its mobile and embedded devices perks, Tensorflow Lite's file size reduction is very advantageous when you have an autoscaled model deployment, like an [HPA in Kubernetes](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale), which dynamically increases the number of available pods when you have higher workloads. When it comes to file size, you have two options: either you use mountable volumes, which can be only used by one node at a time, or you require the model be downloaded every time you auto scale. Imagine having to download 2, 3 or 10 gigs at a time, every time you spin up a new instance. It's so inefficient!

The first step to using TensorFlow Lite is model conversion (turning that `.tf` file into a `.tflite`), but I'll skip this step here since it requires another blog post all together. You can find more details on how to convert your regular Tensorflow model to Tensorflow Lite [here](https://www.tensorflow.org/lite/convert). My main interest is discussing the specifics of deploying the model using Tensorflow Lite.

Second, which is the core benefit of using TensorFlow Lite is the ability to ship your models to iOS and Android devices. If you read the documentation closely, you will notice that the whole API was designed to serve that purpose. Take the `.invoke()` method after you load the interpreter:

A TensorFlow Lite model in Python might look like

{{< highlight python >}}

import numpy as np
import tensorflow as tf

class LukeSkywalker:
    def __init__(self, model_path: str = "model.tflite") -> None:
        self.interpreter = tf.lite.Interpreter(model_path)
        self.interpreter.allocate_tensors()
        self.input_details = self.interpreter.get_input_details()
        self.output_details = self.interpreter.get_output_details()

    def predict(self, input_data: np.array) -> np.array:
        self.interpreter.set_tensor(self.input_details[0]["index"], input_data)
        self.interpreter.invoke()
        return np.squeeze(self.interpreter.get_tensor(self.output_details[0]["index"])[0])

{{< /highlight >}}

The Python example is pretty straightforward and concise. If you wanted to do it in Swift, you'd do whatever those guys do with MVC patterns:

{{< highlight swift >}}

init?(modelFileInfo: FileInfo, labelsFileInfo: FileInfo, threadCount: Int = 1) {
 ...
  self.threadCount = threadCount
  var options = InterpreterOptions()
  options.threadCount = threadCount
  options.isErrorLoggingEnabled = true
  do {
    interpreter = try Interpreter(modelPath: modelPath, options: options)
  } catch let error {
    print("Failed to create the interpreter with error: \(error.localizedDescription)")
    return nil
  }

{{< /highlight >}}

Notice that you can specify thread count for the model and you can also delegate to [CoreML](https://www.tensorflow.org/lite/performance/coreml_delegate) if you have that enabled in your Swift project.

The main caveat here is that Tensorflow Lite is threaded by default and you need to either define the number of threads you want to use, or just set it to `0`. But even then you can run into another problem if you don't know what resources are available to you. For example, I use spot instances pools on AWS and those are varied, with different kinds of Intel processors, which makes it hard to know if I should thread, and if so, how much.

Let's try it anyways, an example of setting the number of threads in python:

{{< highlight python >}}
import os

import numpy as np
import tensorflow as tf

os.environ["TF_NUM_INTEROP_THREADS"] = "1"
os.environ["TF_NUM_INTRAOP_THREADS"] = "4"

class LukeSkywalker:
...
{{< /highlight >}}

To give some more context around the purpose of these environment variables:

`TF_NUM_INTEROP_THREADS`: Determines the number of threads used by independent non-blocking operations. 0 means the system picks an appropriate number.

`TF_NUM_INTRAOP_THREADS`: Certain operations like matrix multiplication and reductions can utilize parallel threads for speed ups. A value of 0 means the system picks an appropriate number.

I spent an entire week of my life trying to fine tune an "appropriate number" where I could maintain high performance. While researching recommendations from the Tensorflow community, everyone said the same thing: always set both inter and intraop thread counts to `1`. This [`djl`](https://djl.ai/docs/development/inference_performance_optimization.html) library is just one of many places I saw the recommendation.
I found this really frustrating. If you benchmark for model speed, we are talking in some cases `700 milliseconds` per inference when `TF_NUM_INTRAOP_THREADS=1` vs `200 milliseconds` per inference when `TF_NUM_INTRAOP_THREADS=4` (This will vary by what kind of model). So why in the world would anyone just use `1`?

So I ignored the recommendation, and I decided to test my model on a 4-node kubernetes cluster and set `TF_NUM_INTRAOP_THREADS` to `4`. What I quickly noticed was the nodes started running at `400%` CPU utilization, which I attributed to AWS using a standard commodity `t3.xlarge` 4-core machine. Then I ran into an even bigger problem, since the HPA autoscaling I'd set up got triggered, thus creating 3 more pods with the same model, all running at that same high utilization rate. This left me with four pods, all running at 400% utilization, which result cannibalization in system resources. This was particularly distressing, since I'd started this whole endeavor thinking I was going to get faster results!

Being the crafty Python developer that I am, I decided to try setting my thread count to `TF_NUM_INTRAOP_THREADS=1` while also using `ThreadPoolExecutor`, maybe that will help (GIL, says no)

{{< highlight python >}}
import concurrent.futures
import os

import numpy as np
import tensorflow as tf

os.environ["TF_NUM_INTEROP_THREADS"] = "1"
os.environ["TF_NUM_INTRAOP_THREADS"] = "1"

class LukeSkywalker:
...

if __name__ == "__main__":
    predictor = LukeSkywalker()
    with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
        future = executor.map(
            predictor.predict, [np.array([0, 0, 0]), np.array([0, 0, 0])]
        )

    print(future.result())

{{< /highlight >}}

This resulted in thread-safe errors, because you can't share `.invoke()` methods across threads. Another dead end! Time to reverse course to a different approach for a different blog post.

And so goes the cautionary tale of using TensorFlow Lite to deploy your models to production.
Hopefully you now know a bit more about the scalability issues you can run into, and know how to avoid them.

You can build the best model man has ever seen, but if you can't scale it properly so it can be utilized in a timely manner, the model becomes obsolete.
Instead of fighting with thread counts, invest some time thinking about how your model is going to evolve over time, and what scale requirements you have before saying "I'm going to use BERT with `max_token_length=1024`."

### References

- [https://www.tensorflow.org/lite](https://www.tensorflow.org/lite)
- [https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale)