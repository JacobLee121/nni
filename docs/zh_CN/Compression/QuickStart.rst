Quick Start
===========

..  toctree::
    :hidden:

    Tutorial <Tutorial>


Model compression usually consists of three stages: 1) pre-training a model, 2) compress the model, 3) fine-tuning the model. NNI mainly focuses on the second stage and provides very simple APIs for compressing a model. Follow this guide for a quick look at how easy it is to use NNI to compress a model. 

Model Pruning
-------------

Here we use `level pruner <../Compression/Pruner.rst#level-pruner>`__ as an example to show the usage of pruning in NNI.

Step1. 编写配置
^^^^^^^^^^^^^^^^^^^^^^^^^^

编写配置来指定要剪枝的层。 The following configuration means pruning all the ``default``\ ops to sparsity 0.5 while keeping other layers unpruned.

.. code-block:: python

   config_list = [{
       'sparsity': 0.5,
       'op_types': ['default'],
   }]

The specification of configuration can be found `here <./Tutorial.rst#specify-the-configuration>`__. 注意，不同的 Pruner 可能有自定义的配置字段，例如，AGP Pruner 有 ``start_epoch``。 详情参考每个 Pruner 的 `用法 <./Pruner.rst>`__，来调整相应的配置。

Step2. Choose a pruner and compress the model
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

首先，使用模型来初始化 Pruner，并将配置作为参数传入，然后调用 ``compress()`` 来压缩模型。 Note that, some algorithms may check gradients for compressing, so we also define an optimizer and pass it to the pruner.

.. code-block:: python

   from nni.algorithms.compression.pytorch.pruning import LevelPruner

   optimizer_finetune = torch.optim.SGD(model.parameters(), lr=0.01)
   pruner = LevelPruner(model, config_list, optimizer_finetune)
   model = pruner.compress()

然后，使用正常的训练方法来训练模型 （如，SGD），剪枝在训练过程中是透明的。 Some pruners (e.g., L1FilterPruner, FPGMPruner) prune once at the beginning, the following training can be seen as fine-tune. Some pruners (e.g., AGPPruner) prune your model iteratively, the masks are adjusted epoch by epoch during training.

Note that, ``pruner.compress`` simply adds masks on model weights, it does not include fine-tuning logic. 如果要想调优压缩后的模型，需要在 ``pruner.compress`` 后增加调优的逻辑。

For example:

.. code-block:: python

   for epoch in range(1, args.epochs + 1):
        pruner.update_epoch(epoch)
        train(args, model, device, train_loader, optimizer_finetune, epoch)
        test(model, device, test_loader)

More APIs to control the fine-tuning can be found `here <./Tutorial.rst#apis-to-control-the-fine-tuning>`__. 


Step3. 导出压缩结果
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

After training, you can export model weights to a file, and the generated masks to a file as well. 也支持导出 ONNX 模型。

.. code-block:: python

   pruner.export_model(model_path='pruned_vgg19_cifar10.pth', mask_path='mask_vgg19_cifar10.pth')

Plese refer to :githublink:`mnist example <examples/model_compress/pruning/naive_prune_torch.py>` for example code.

More examples of pruning algorithms can be found in :githublink:`basic_pruners_torch <examples/model_compress/pruning/basic_pruners_torch.py>` and :githublink:`auto_pruners_torch <examples/model_compress/pruning/auto_pruners_torch.py>`.


Model Quantization
------------------

Here we use `QAT  Quantizer <../Compression/Quantizer.rst#qat-quantizer>`__ as an example to show the usage of pruning in NNI.

Step1. Write configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

   config_list = [{
       'quant_types': ['weight'],
       'quant_bits': {
           'weight': 8,
       }, # you can just use `int` here because all `quan_types` share same bits length, see config for `ReLu6` below.
       'op_types':['Conv2d', 'Linear']
   }, {
       'quant_types': ['output'],
       'quant_bits': 8,
       'quant_start_step': 7000,
       'op_types':['ReLU6']
   }]

The specification of configuration can be found `here <./Tutorial.rst#quantization-specific-keys>`__.

Step2. Choose a quantizer and compress the model
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

   from nni.algorithms.compression.pytorch.quantization import QAT_Quantizer

   quantizer = QAT_Quantizer(model, config_list)
   quantizer.compress()


Step3. Export compression result
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

您可以使用 ``torch.save`` api 直接导出量化模型。量化后的模型可以通过 ``torch.load`` 加载，不需要做任何额外的修改。

.. code-block:: python

   # 保存使用 NNI QAT 算法生成的量化模型
   torch.save(model.state_dict(), "quantized_model.pth")

Plese refer to :githublink:`mnist example <examples/model_compress/quantization/QAT_torch_quantizer.py>` for example code.

Congratulations! You've compressed your first model via NNI. To go a bit more in depth about model compression in NNI, check out the `Tutorial <./Tutorial.rst>`__.