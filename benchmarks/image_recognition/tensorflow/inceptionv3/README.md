# Inception V3

This document has instructions for how to run Inception V3 for the
following modes/platforms:
* [Int8 inference](#int8-inference-instructions)
* [FP32 inference](#fp32-inference-instructions)

Benchmarking instructions and scripts for model training and inference
other platforms are coming later.

## Int8 Inference Instructions

1. Clone this [intelai/models](https://github.com/IntelAI/models)
repository:

```
$ git clone git@github.com:IntelAI/models.git
```

This repository includes launch scripts for running benchmarks and the
an optimized version of the inceptionv3 model code.

2. Clone the [tensorflow/models](https://github.com/tensorflow/models)
repository:

```
git clone git@github.com:tensorflow/models.git
```

This repository is used for dependencies that the Inception V3 model
requires.

3. Download the pre-trained inceptionv3 model:

```
$ wget https://storage.cloud.google.com/intel-optimized-tensorflow/models/inceptionv3_int8_pretrained_model.pb
```

4. Build a docker image using master of the official
[TensorFlow](https://github.com/tensorflow/tensorflow) repository with
`--config=mkl`. More instructions on
[how to build from source](https://software.intel.com/en-us/articles/intel-optimization-for-tensorflow-installation-guide#inpage-nav-5).

5. If you would like to run Inception V3 inference and test for
accuracy, you will need the ImageNet dataset. Benchmarking for latency
and throughput do not require the ImageNet dataset.

Register and download the
[ImageNet dataset](http://image-net.org/download-images).

Once you have the raw ImageNet dataset downloaded, we need to convert
it to the TFRecord format. This is done using the
[build_imagenet_data.py](https://github.com/tensorflow/models/blob/master/research/inception/inception/data/build_imagenet_data.py)
script. There are instructions in the header of the script explaining
its usage.

After the script has completed, you should have a directory with the
sharded dataset something like:

```
$ ll /home/myuser/datasets/ImageNet_TFRecords
-rw-r--r--. 1 user 143009929 Jun 20 14:53 train-00000-of-01024
-rw-r--r--. 1 user 144699468 Jun 20 14:53 train-00001-of-01024
-rw-r--r--. 1 user 138428833 Jun 20 14:53 train-00002-of-01024
...
-rw-r--r--. 1 user 143137777 Jun 20 15:08 train-01022-of-01024
-rw-r--r--. 1 user 143315487 Jun 20 15:08 train-01023-of-01024
-rw-r--r--. 1 user  52223858 Jun 20 15:08 validation-00000-of-00128
-rw-r--r--. 1 user  51019711 Jun 20 15:08 validation-00001-of-00128
-rw-r--r--. 1 user  51520046 Jun 20 15:08 validation-00002-of-00128
...
-rw-r--r--. 1 user  52508270 Jun 20 15:09 validation-00126-of-00128
-rw-r--r--. 1 user  55292089 Jun 20 15:09 validation-00127-of-00128
```

6. Next, navigate to the `benchmarks` directory in your local clone of
the [intelai/models](https://github.com/IntelAI/models) repo from step 1.
The `launch_benchmark.py` script in the `benchmarks` directory is
used for starting a benchmarking run in a optimized TensorFlow docker
container. It has arguments to specify which model, framework, mode,
platform, and docker image to use, along with your path to the ImageNet
TF Records that you generated in step 5.

Substitute in your own `--data-location` (from step 5, for accuracy
only), `--in-graph` pretrained model file path (from step 3),
`--model-source-dir` for the location where you cloned the
[tensorflow/models](https://github.com/tensorflow/models) repo
(from step 2), and the name/tag for your docker image (from step 4).

Inception V3 can be run for accuracy, latency benchmarking, or throughput
benchmarking. Use one of the following examples below, depending on
your use case.

For accuracy (using your `--data-location`, `--accuracy-only` and
`--batch-size 128`):

```
python launch_benchmark.py \
    --model-name inceptionv3 \
    --platform int8 \
    --mode inference \
    --framework tensorflow \
    --accuracy-only \
    --batch-size 128 \
    --docker-image tf_int8_docker_image \
    --in-graph /home/myuser/inceptionv3_int8_pretrained_model.pb \
    --data-location /home/myuser/datasets/ImageNet_TFRecords \
    -- input_height=299 input_width=299
```

For latency (using `--benchmark-only`, `--single-socket` and `--batch-size 1`):

```
python launch_benchmark.py \
    --model-name inceptionv3 \
    --platform int8 \
    --mode inference \
    --framework tensorflow \
    --benchmark-only \
    --batch-size 1 \
    --single-socket \
    --docker-image tf_int8_docker_image \
    --in-graph /home/myuser/inceptionv3_int8_pretrained_model.pb \
    --data-location /home/myuser/datasets/ImageNet_TFRecords \
    -- input_height=299 input_width=299
```

For throughput (using `--benchmark-only`, `--single-socket` and `--batch-size 128`):

```
python launch_benchmark.py \
    --model-name inceptionv3 \
    --platform int8 \
    --mode inference \
    --framework tensorflow \
    --benchmark-only \
    --batch-size 128 \
    --single-socket \
    --docker-image tf_int8_docker_image \
    --in-graph /home/myuser/inceptionv3_int8_pretrained_model.pb \
    --data-location /home/myuser/datasets/ImageNet_TFRecords \
    -- input_height=299 input_width=299
```

Note that the `--verbose` flag can be added to any of the above commands
to get additional debug output.

7. The log file is saved to the
`models/benchmarks/common/tensorflow/logs` directory. Below are
examples of what the tail of your log file should look like for the
different configs.

Example log tail when running for accuracy:

```
Processed 49920 images. (Top1 accuracy, Top5 accuracy) = (0.7662, 0.9335)
lscpu_path_cmd = command -v lscpu
lscpu located here: /usr/bin/lscpu
Executing command: python /workspace/intelai_models/int8/accuracy.py --input_height=299 --input_width=299 --num_intra_threads=56 --num_inter_threads=2 --batch_size=128 --input_graph=/in_graph/final_int8_inceptionv3.pb --data_location=/dataset
Ran inference with batch size 128
Log location outside container: /home/myuser/models/benchmarks/common/tensorflow/logs/benchmark_inceptionv3_inference.log
```

Example log tail when benchmarking for latency:
```
[Running warmup steps...]
steps = 10, 56.7987541472 images/sec
[Running benchmark steps...]
steps = 10, 56.853417193 images/sec
steps = 20, 56.4623275224 images/sec
steps = 30, 54.947453919 images/sec
steps = 40, 56.507207717 images/sec
steps = 50, 56.6759543274 images/sec
lscpu_path_cmd = command -v lscpu
lscpu located here: /usr/bin/lscpu
Executing command: numactl --cpunodebind=0 --membind=0 python /workspace/intelai_models/int8/benchmark.py --input_height=299 --input_width=299 --warmup_steps=10 --num_intra_threads=56 --num_inter_threads=2 --batch_size=1 --input_graph=/in_graph/final_int8_inceptionv3.pb --steps=50
Ran inference with batch size 1
Log location outside container: /home/myuser/models/benchmarks/common/tensorflow/logs/benchmark_inceptionv3_inference.log
```

Example log tail when benchmarking for throughput:
```
[Running warmup steps...]
steps = 10, 336.523181805 images/sec
[Running benchmark steps...]
steps = 10, 330.464868432 images/sec
steps = 20, 337.603490289 images/sec
steps = 30, 337.37478909 images/sec
steps = 40, 335.896171239 images/sec
steps = 50, 337.467250577 images/sec
lscpu_path_cmd = command -v lscpu
lscpu located here: /usr/bin/lscpu
Executing command: numactl --cpunodebind=0 --membind=0 python /workspace/intelai_models/int8/benchmark.py --input_height=299 --input_width=299 --warmup_steps=10 --num_intra_threads=56 --num_inter_threads=2 --batch_size=128 --input_graph=/in_graph/final_int8_inceptionv3.pb --steps=50
Ran inference with batch size 128
Log location outside container: /home/myuser/models/benchmarks/common/tensorflow/logs/benchmark_inceptionv3_inference.log
```

## FP32 Inference Instructions

1. Clone this [intelai/models](https://github.com/IntelAI/models)
repository:

```
$ git clone git@github.com:IntelAI/models.git
```

2. Download the pre-trained inceptionv3 model:

```
$ wget https://storage.cloud.google.com/intel-optimized-tensorflow/models/inceptionv3_fp32_pretrained_model.pb
```

3. Navigate to the `benchmarks` directory in your local clone of
the [intelai/models](https://github.com/IntelAI/models) repo from step 1.
The `launch_benchmark.py` script in the `benchmarks` directory is
used for starting a benchmarking run in a optimized TensorFlow docker
container. It has arguments to specify which model, framework, mode,
platform, and docker image.

Substitute in your own `--in-graph` pretrained model file path (from step 2).

Inception V3 can be run for latency benchmarking, or throughput
benchmarking. Use one of the following examples below, depending on
your use case.

* For latency (using `--batch-size 1`):

```
python launch_benchmark.py \
    --model-name inceptionv3 \
    --platform fp32 \
    --mode inference \
    --framework tensorflow \
    --batch-size 1 \
    --single-socket \
    --verbose \
    --docker-image intelaipg/intel-optimized-tensorflow:latest-devel-mkl \
    --in-graph /home/myuser/inceptionv3_fp32_pretrained_model.pb
```
Example log tail when benchmarking for latency:
```
Inference with dummy data.
Iteration 1: 1.096 sec
Iteration 2: 0.014 sec
Iteration 3: 0.014 sec
Iteration 4: 0.015 sec
...
Iteration 36: 0.014 sec
Iteration 37: 0.014 sec
Iteration 38: 0.014 sec
Iteration 39: 0.014 sec
Iteration 40: 0.014 sec
Average time: 0.014 sec
Batch size = 1
Latency: 14.434 ms
Throughput: 69.283 images/sec
lscpu_path_cmd = command -v lscpu
lscpu located here: /usr/bin/lscpu
Received these standard args: Namespace(accuracy_only=False, batch_size=1, benchmark_only=False, checkpoint=None, data_location=None, framework='tensorflow', input_graph='/in_graph/inceptionv3_fp32_pretrained_model.pb', intelai_models='/workspace/intelai_models', mode='inference', model_args=[], model_name='inceptionv3', model_source_dir=None, num_cores=-1, num_inter_threads=1, num_intra_threads=28, platform='fp32', single_socket=True, socket_id=0, use_case='image_recognition', verbose=True)
Received these custom args: []
Executing command: numactl --cpunodebind=0 --membind=0 python /workspace/intelai_models/fp32/eval_image_classifier_inference.py --input-graph=/in_graph/inceptionv3_fp32_pretrained_model.pb --model-name=inceptionv3 --inter-op-parallelism-threads=1 --intra-op-parallelism-threads=28 --batch-size=1
PYTHONPATH: :/workspace/intelai_models
RUNCMD: python common/tensorflow/run_tf_benchmark.py     --framework=tensorflow     --use-case=image_recognition     --model-name=inceptionv3     --platform=fp32     --mode=inference     --intelai-models=/workspace/intelai_models     --batch-size=1     --single-socket     --verbose     --in-graph=/in_graph/inceptionv3_fp32_pretrained_model.pb 
Batch Size: 1
Ran inference with batch size 1
Log location outside container: /home/myuser/logs/benchmark_inceptionv3_inference.log
```

* For throughput (using `--batch-size 128`):

```
python launch_benchmark.py \
    --model-name inceptionv3 \
    --platform fp32 \
    --mode inference \
    --framework tensorflow \
    --batch-size 128 \
    --single-socket \
    --verbose \
    --docker-image intelaipg/intel-optimized-tensorflow:latest-devel-mkl \
    --in-graph /home/myuser/inceptionv3_fp32_pretrained_model.pb
```
Example log tail when benchmarking for throughput:
```
Inference with dummy data.
Iteration 1: 1.969 sec
Iteration 2: 0.785 sec
Iteration 3: 0.770 sec
Iteration 4: 0.750 sec
...
Iteration 36: 0.754 sec
Iteration 37: 0.754 sec
Iteration 38: 0.754 sec
Iteration 39: 0.749 sec
Iteration 40: 0.750 sec
Average time: 0.750 sec
Batch size = 128
Throughput: 170.579 images/sec
lscpu_path_cmd = command -v lscpu
lscpu located here: /usr/bin/lscpu
Received these standard args: Namespace(accuracy_only=False, batch_size=128, benchmark_only=False, checkpoint=None, data_location=None, framework='tensorflow', input_graph='/in_graph/inceptionv3_fp32_pretrained_model.pb', intelai_models='/workspace/intelai_models', mode='inference', model_args=[], model_name='inceptionv3', model_source_dir=None, num_cores=-1, num_inter_threads=1, num_intra_threads=28, platform='fp32', single_socket=True, socket_id=0, use_case='image_recognition', verbose=True)
Received these custom args: []
Executing command: numactl --cpunodebind=0 --membind=0 python /workspace/intelai_models/fp32/eval_image_classifier_inference.py --input-graph=/in_graph/inceptionv3_fp32_pretrained_model.pb --model-name=inceptionv3 --inter-op-parallelism-threads=1 --intra-op-parallelism-threads=28 --batch-size=128
PYTHONPATH: :/workspace/intelai_models
RUNCMD: python common/tensorflow/run_tf_benchmark.py     --framework=tensorflow     --use-case=image_recognition     --model-name=inceptionv3     --platform=fp32     --mode=inference     --intelai-models=/workspace/intelai_models     --batch-size=128     --single-socket     --verbose     --in-graph=/in_graph/inceptionv3_fp32_pretrained_model.pb
Batch Size: 128
Ran inference with batch size 128
Log location outside container: /home/myuser/logs/benchmark_inceptionv3_inference.log
```