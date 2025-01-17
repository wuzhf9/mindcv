# Inference Service Deployment

MindSpore Serving is a lightweight and high-performance service module that helps MindSpore developers efficiently deploy online inference services in the production environment. After completing model training on MindSpore, you can export the MindSpore model and use MindSpore Serving to create an inference service for the model.

This tutorial uses mobilenet_v2_100 network as an example to describe how to deploy the Inference Service based on MindSpore Serving.

## Environment Preparation

Before deploying, ensure that MindSpore Serving has been properly installed and the environment variables are configured. To install and configure MindSpore Serving on your PC, go to the [MindSpore Serving installation page](https://www.mindspore.cn/serving/docs/en/master/serving_install.html).

## Exporting the Model

To implement cross-platform or hardware inference (e.g., Ascend AI processor, MindSpore device side, GPU, etc.), the model file of MindIR format should be generated by network definition and CheckPoint. In MindSpore, the function of exporting the network model is `export`, and the main parameters are as follows:

- `net`: MindSpore network structure.
- `inputs`: Network input, and the supported input type is `Tensor`. If multiple values are input, the values should be input at the same time, for example, `ms.export(network, ms.Tensor(input1), ms.Tensor(input2), file_name='network', file_format='MINDIR')`.
- `file_name`: Name of the exported model file. If `file_name` doesn't contain the corresponding suffix (for example, .mindir), the system will automatically add one after `file_format` is set.
- `file_format`: MindSpore currently supports ‘AIR’, ‘ONNX’ and ‘MINDIR’ format for exported model.

The following code uses mobilenet_v2_100 as an example to export the pretrained network model of MindCV and obtain the model file in MindIR format.

```python
from mindcv.models import create_model
import numpy as np
import mindspore as ms

model = create_model(model_name='mobilenet_v2_100_224', num_classes=1000, pretrained=True)

input_np = np.random.uniform(0.0, 1.0, size=[1, 3, 224, 224]).astype(np.float32)

# Export mobilenet_v2_100_224.mindir to current folder.
ms.export(model, ms.Tensor(input_np), file_name='mobilenet_v2_100_224', file_format='MINDIR')
```

## Deploying the Serving Inference Service

### Configuring the Service

Start Serving with the following files:

```text
demo
├── mobilenet_v2_100_224
│   ├── 1
│   │   └── mobilenet_v2_100_224.mindir
│   └── servable_config.py
│── serving_server.py
├── serving_client.py
├── imagenet1000_clsidx_to_labels.txt
└── test_image
    ├─ dog
    │   ├─ dog.jpg
    │   └─ ……
    └─ ……
```

- `mobilenet_v2_100_224`: Model folder. The folder name is the model name.
- `mobilenet_v2_100_224.mindir`: Model file generated by the network in the previous step, which is stored in folder 1 (the number indicates the version number). Different versions are stored in different folders. The version number must be a string of digits. By default, the latest model file is started.
- `servable_config.py`: Model configuration script. Declare the model and specify the input and output parameters of the model.
- `serving_server.py`: Script to start the Serving server.
- `serving_client.py`: Script to start the Python client.
- `imagenet1000_clsidx_to_labels.txt`: Index of 1000 labels for the ImageNet dataset, available at [examples/data/](https://github.com/mindspore-lab/mindcv/tree/main/examples/data).
- `test_image`: Test images, available at [README](https://github.com/mindspore-lab/mindcv/blob/main/README.md).

Content of the configuration file `servable_config.py`:

```python
from mindspore_serving.server import register

# Declare the model. The parameter model_file indicates the name of the model file, and model_format indicates the model type.
model = register.declare_model(model_file="mobilenet_v2_100_224.mindir", model_format="MindIR")

# The input parameters of the Servable method are specified by the input parameters of the Python method. The output parameters of the Servable method are specified by the output_names of register_method.
@register.register_method(output_names=["score"])
def predict(image):
    x = register.add_stage(model, image, outputs_count=1)
    return x
```

### Starting the Service

The `server` function of MindSpore can provide deployment service through either gRPC or RESTful. The following uses gRPC as an example. The service startup script `serving_server.py` deploys the `mobilenet_v2_100_224` in the local directory to device 0 and starts the gRPC server at 127.0.0.1:5500. Content of the script:

```python
import os
import sys
from mindspore_serving import server

def start():
    servable_dir = os.path.dirname(os.path.realpath(sys.argv[0]))

    servable_config = server.ServableStartConfig(servable_directory=servable_dir, servable_name="mobilenet_v2_100_224",
                                                 device_ids=0)
    server.start_servables(servable_configs=servable_config)
    server.start_grpc_server(address="127.0.0.1:5500")

if __name__ == "__main__":
    start()
```

If the following log information is displayed on the server, the gRPC service is started successfully.

```text
Serving gRPC server start success, listening on 127.0.0.1:5500
```

### Inference Execution

Start the Python client using `serving_client.py`. The client script uses the `create_transforms`, `create_dataset` and `create_loader` functions of `mindcv.data` to preprocess the image and send the image to the serving server. Then postprocesse the result returned by the server and prints the prediction label of the image.

```python
import os
from mindspore_serving.client import Client
import numpy as np
from mindcv.data import create_transforms, create_dataset, create_loader

num_workers = 1

# Dataset directory path
data_dir = "./test_image/"

dataset = create_dataset(root=data_dir, split='', num_parallel_workers=num_workers)
transforms_list = create_transforms(dataset_name='ImageNet', is_training=False)
data_loader = create_loader(
    dataset=dataset,
    batch_size=1,
    is_training=False,
    num_classes=1000,
    transform=transforms_list,
    num_parallel_workers=num_workers
)
with open("imagenet1000_clsidx_to_labels.txt") as f:
    idx2label = eval(f.read())

def postprocess(score):
    max_idx = np.argmax(score)
    return idx2label[max_idx]

def predict():
    client = Client("127.0.0.1:5500", "mobilenet_v2_100_224", "predict")
    instances = []
    images, _ = next(data_loader.create_tuple_iterator())
    image_np = images.asnumpy().squeeze()
    instances.append({"image": image_np})
    result = client.infer(instances)

    for instance in result:
        label = postprocess(instance["score"])
        print(label)

if __name__ == '__main__':
    predict()
```

If the following information is displayed, Serving service has correctly executed the inference of the mobilenet_v2_100 model:
```text
Labrador retriever
```
