# Efficient Machine Learning: Optimizations and Benchmarking
Deep learning is becoming popular nowadays. It is being used in different applications like classification, segmentation, pose estimation, augmented reality and self-driving cars. Primary goal in deep learning applications is to achieve accuracy. This can be done using big models but these complex models leads several issues in real time applications. These real-time applications run on edge devices but large models put constraints on memory and latency.

TensorFlow Lite benchmark tools currently measure and calculate statistics for the following important performance metrics:
* Initialization time
* Inference time of warmup state
* Inference time of steady state
* Memory usage during initialization time
* Overall memory usage

## Description of Optimization
### Magnitude-based weight pruning
Gradually zeroes out model weights during the training process to achieve model sparsity. Sparse models are easier to compress, and we can skip the zeroes during inference for latency improvements. In unstructured pruning, individual weight connections are removed from a network by setting them to 0. In structured pruning, groups of weight connections are removed together, such as entire channels or filters. Unfortunately, structured pruning severely limits the maximum sparsity that limits both the performance and memory. improvements.

### Clustering
Clustering, also known as weight sharing, helps make models more efficient for deployment by reducing the number of different weight values they contain. The process involves grouping the weights of each layer into clusters. Within each cluster, all the weights share the same value, known as the cluster's centroid value.
![Picture7](https://github.com/alishafique3/Efficient-Machine-Learning_Optimizations-and-Benchmarking/assets/17300597/8354eded-8734-4937-96f1-62d27f7da053)


### Quantization
Quantization in machine learning refers to reducing the precision or range of numerical values used to represent a model's parameters or weights. It compresses models, making them more memory-efficient and faster to execute. It involves mapping high-precision values to lower-precision representations and can be done through post-training quantization, quantization-aware training, or a combination of both. Quantization is beneficial for deploying models on resource-constrained devices, as it reduces memory and computational requirements while maintaining acceptable accuracy.
![Picture6](https://github.com/alishafique3/Efficient-Machine-Learning_Optimizations-and-Benchmarking/assets/17300597/1ea4a90f-2308-4ecc-928f-d70856654eb8)

## Methodology
Base code for model training and dataset is provided in the [blog](https://www.pyimagesearch.com/2019/11/18/fire-and-smoke-detection-with-keras-and-deep-learning/) at Pyimagesearch by Adrian Rosebrock on November 18, 2019. 
This project has contributed to the following: 
1.	Model training is modified for pruning, clustering, and collaborative optimizations.
2.	Post-Training optimization
3.	Benchmarking in Android devices

![Picture4](https://github.com/alishafique3/Efficient-Machine-Learning_Optimizations-and-Benchmarking/assets/17300597/944f3821-2b1a-4b18-b820-ff8909bc8561)



### Step1: Loading and preprocessing of the dataset
A dataset can be accessed either by uploading it in the drive of google colab or google drive. In this project, the dataset has been uploaded on google drive and it is accessed in the code by mounting google drive in Colab. 

### Step2: Model architecture and training
A deep neural network is used for classification. It contains convolutional layers, max pooling, and batch normalization layers. Adam optimizer is used for training with binary cross entropy loss function. It is trained for 50 epochs. Four different models are trained with different optimizations.

![Picture5](https://github.com/alishafique3/Efficient-Machine-Learning_Optimizations-and-Benchmarking/assets/17300597/3b65816f-2b3a-4770-ba23-3832e8f8ee7f)

#### Unoptimized Baseline Model:
In this model, training is done with no optimization. This model is used to compare with optimized versions. This model is converted into TensorLite format. 
Baseline model is developed using following code:
```python
converter1 = tf.lite.TFLiteConverter.from_keras_model(model)
baseline_tflite_model = converter1.convert()
```

#### Dynamic range quantization Model:
In this model, training is done without any optimization, while model is converted with dynamic range quantization during tensorlite conversion process. 
```python
converter2 = tf.lite.TFLiteConverter.from_keras_model(model)
converter2.optimizations = [tf.lite.Optimize.DEFAULT]
quantized_tflite_model = converter2.convert()
```

#### Float16 quantization Model:
In this model, training is done without any optimization, while model is converted with float16 quantization during tensorlite conversion process. 
```python
converter3 = tf.lite.TFLiteConverter.from_keras_model(model)
converter3.optimizations = [tf.lite.Optimize.DEFAULT]
converter3.target_spec.supported_types = [tf.float16]
fquantized_tflite_model = converter3.convert()
```

#### Clustered and Quantized Model:
In this model, training is done with regular model then fine tuning is done with clustering optimization. In the end, the model is converted with dynamic range quantization during tensorlite conversion process.

```python
#Fine Tunning after regular training
cluster_weights = tfmot.clustering.keras.cluster_weights
CentroidInitialization = tfmot.clustering.keras.CentroidInitialization
clustering_params = {
  'number_of_clusters': 8,
  'cluster_centroids_init': CentroidInitialization.KMEANS_PLUS_PLUS,
  'cluster_per_channel': True,
}
clustered_model = cluster_weights(model, **clustering_params)
# Use smaller learning rate for fine-tuning
opt = tf.keras.optimizers.Adam(learning_rate=1e-5)
clustered_model.compile(loss="binary_crossentropy", optimizer=opt,
  metrics=["accuracy"])
clustered_model.summary()
```
After Fine-tuning: 

After fine-tunning we need to remove clustering tf.variables that are needed for clustering otherwise it will increase the model size.
```python 
stripped_clustered_model = tfmot.clustering.keras.strip_clustering(clustered_model)
```
After clustering we will again apply post-training quantization to get quantized model.
```python
Converter4 = tf.lite.TFLiteConverter.from_keras_model(stripped_clustered_model)
Converter4.optimizations = [tf.lite.Optimize.DEFAULT]
quantized_tflite_model = converter4.convert()
```
#### Pruning and Quantized Model
In this model, training is done with regular model then fine tuning is done with pruning optimization. In the end, the model is converted with dynamic range quantization during tensorlite conversion process. 
After Training, Fine Tuning stage:
```python
prune_low_magnitude = tfmot.sparsity.keras.prune_low_magnitude
pruning_params = {
      'pruning_schedule': tfmot.sparsity.keras.ConstantSparsity(0.5, begin_step=0, frequency=100)
  }
callbacks = [
  tfmot.sparsity.keras.UpdatePruningStep()
]

pruned_model = prune_low_magnitude(model, **pruning_params)

# Use smaller learning rate for fine-tuning
opt = tf.keras.optimizers.Adam(learning_rate=1e-5)

pruned_model.compile(loss="binary_crossentropy", optimizer=opt,
  metrics=["accuracy"])
 ```
After fine-tunning we need to remove pruning tf.variables that are needed for clustering otherwise it will increase the model size.
```python
stripped_pruned_model = tfmot.sparsity.keras.strip_pruning(pruned_model)
```
After clustering we will again apply post-training quantization to get quantized model.
```python
converter = tf.lite.TFLiteConverter.from_keras_model(stripped_pruned_model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
pqat_tflite_model = converter.convert()
```

### Step3: Benchmarking on Android Device
ADB can be installed in windows and linux. I have found these videos very useful to setup [Link](https://www.youtube.com/watch?v=26GI3z6tI3E). and get familiarize with ADB [Link](https://www.youtube.com/watch?v=uOPcUjVl2YQ). Once ADB is setup, mobile can be controlled with computer with ADB commands via USB or wifi. Remember android device should be in USB Debugging mode or wireless debugging mode with USB or wifi connection respectively. Android emulator can also be used for benchmarking the models. Android emulator can be found using android studio. (Android studio has both emulator and ADB)
`adb devices` will list all android devices/emulator connected with computer if it does not work or if it shows offline, try to connect android again kill ADB connection using `adb kill-server` and try again using `adb devices`

Once android device connection with computer is established, benchmarking file and target TFLite models will be sent to android device using ADB commands. There are two options of using the benchmark tool with Android. 
1.	Native benchmark binary 
2.	Android benchmark app, a better gauge of how the model would perform in the app.

Both benchmarking files are available in android_aarch64 and android_arm at [Link](https://www.tensorflow.org/lite/performance/measurement). To find your android device specific architecture, you can download droid app from playstore.
#### Android benchmark app 
once android benchmarking file is downloaded, keep TFLite models and Android benchmark app in same folder and open terminal in that folder, start following lines
```python
adb devices
adb install -r -d -g android_aarch64_benchmark_model.apk  # for android_aarch64 benchmarking  file
adb push mobilenet.tflite /data/local/tmp # for model  file
adb shell am start -S \-n org.tensorflow.lite.benchmark/.BenchmarkModelActivity \--es args '"--graph=/data/local/tmp/mobilenet.tflite \--num_threads=4"'
adb logcat | findstr "Inference timings"
```

#### Native benchmark binary
once android benchmarking file is downloaded, keep TFLite models and Android benchmark app in same folder and open terminal in that folder, start following lines
```python
adb push android_aarch64_benchmark_model /data/local/tmp # for android_aarch64 benchmarking  file
adb shell chmod +x /data/local/tmp/android_aarch64_benchmark_model
adb push mobilenet.tflite /data/local/tmp # for model  file
adb shell /data/local/tmp/android_aarch64_benchmark_model \--graph=/data/local/tmp/mobilenet.tflite \--num_threads=4
```

graph is a required parameter.
*	graph: string 
The path to the TFLite model file.
You can specify more optional parameters for running the benchmark.
*	`num_threads`: int (default=1) The number of threads to use for running TFLite interpreter. A thread is a virtual component that handles the tasks of a CPU core. Multithreading is the ability of the CPU to divide up the work among multiple threads instead of giving it to a single core, to enable concurrent processing. The multiple threads are processed by the different CPU cores in parallel, to speed up performance and save time.
*	`use_gpu`: bool (default=false) you can enable use of GPU-accelerated execution of your models using a delegate. Delegates act as hardware drivers for TensorFlow Lite, allowing you to run the code of your model on GPU processors.
*	`use_nnapi`: bool (default=false) Use Android Neural Networks API (NNAPI) delegate.  It provides acceleration for TensorFlow Lite models on Android devices with supported hardware accelerators including, Graphics Processing Unit (GPU), Digital Signal Processor (DSP) and Neural Processing Unit (NPU).
*	`use_xnnpack`: bool (default=false) Use XNNPACK delegate. XNNPACK is a highly optimized library of neural network inference operators for ARM, x86, and WebAssembly architectures in Android, iOS, Windows, Linux, macOS, and Emscripten environments.
*	`use_hexagon`: bool (default=false) Use Hexagon delegate. This delegate leverages the Qualcomm Hexagon library to execute quantized kernels on the DSP. Note that the delegate is intended to complement NNAPI functionality, particularly for devices where NNAPI DSP acceleration is unavailable

## Result
| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |

## Discussion

## Conclusion
In this project, different optimized models have been compared on android devices. Dynamic quantization plays remarkably well among these optimized models. This project can be extended on different datasets, models and hardware to see the performance of optimization techniques.

## References



