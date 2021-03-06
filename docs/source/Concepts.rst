Core Concepts
#############

Conventions
===========

* ``Model_T`` an instance of :class:`torch.nn.Module`, the teacher model that to be distilled.

* ``Model_S``: an instance of :class:`torch.nn.Module`, the student model, usually smaller than the teacher model for model compression and faster inference speed.

* ``optimizer``: an instance of :class:`torch.optim.Optimizer`.

* ``scheduler``: an instance of a class under :mod:`torch.optim.lr_scheduler`, allows flexible adjustment of learning rate.

* ``dataloader``: data iterator, used to generate data batches. A batch can be a tuple or a dict

    .. code-block:: python

        for batch in dataloader:
            # if batch_postprocessor is not None:
            batch = batch_postprocessor(batch)
            # check batch datatype
            # passes batch to the model and adaptors

    .. note:: 
        1. During training, the distiller will check if the ``batch`` is a ``dict``, if so the model will be called as ``model(**batch, **args)``, otherwise the model is called as ``model(*batch, **args)``. Hence if the batch is not a dict, **users should make sure that the order of each element in the batch is the same as the order of the arguments of** ``model.forward``. ``args`` is used for passing additional parameters.
        2. Users can define a ``batch_postprocessor`` function to post-process batches if needed. ``batch_postprocessor`` should take a batch and return a batch. See the explanation on ``train`` method of :ref:`distillers` for more details.


Configurations
==============

* :class:`~textbrewer.TrainingConfig`: configurations related to general deep learning model training.
* :class:`~textbrewer.DistillationConfig`: configurations related to distillation methods.

Distillers
==========

Distillers are in charge of conducting the actual experiments. The following distillers are available:

* :class:`~textbrewer.BasicDistiller`: **single-teacher single-task** distillation, provides basic distillation strategies.
* :class:`~textbrewer.GeneralDistiller` (Recommended): **single-teacher single-task** distillation, supports intermediate features matching. **Recommended most of the time**.
* :class:`~textbrewer.MultiTeacherDistiller`: **multi-teacher** distillation, which distills multiple teacher models (of the same task) into a single student model. **This class doesn't support Intermediate features matching.**
* :class:`~textbrewer.MultiTaskDistiller`: **multi-task** distillation, which distills multiple teacher models (of different tasks) into a single student. **This class doesn't support Intermediate features matching.**
* :class:`~textbrewer.BasicTrainer`: Supervised training a single model on a labeled dataset, not for distillation. **It can be used to train a teacher model**.

User-Defined Functions
======================

In TextBrewer, there are two functions that should be implemented by users: :func:`callback` and :func:`adaptor` .

.. function:: callback(model, step) -> None

    At each checkpoint, after saving the student model, the `callback` function will be called by the distiller. `callback` can be used to evaluate the performance of the student model at each checkpoint.
    
    .. Note:: 
        If users want to do an evaluation in the callback, remember to add ``model.eval()`` in the callback.

   :param torch.nn.Module model: the student model
   :param int step: the current training step



.. function:: adaptor(batch, model_inputs) -> dict

    It converts the model inputs and outputs to the specified format so that they can be recognized by the distiller. At each training step, batch and model outputs will be passed to the `adaptor`; `adaptor` reorganize the data and returns a dict.
 
    The functionality of the `adaptor` is shown in the figure below:

    .. image:: ../../pics/adaptor.png
        :width: 375px
        :align: center

    :param batch: the input batch to the model
    :param model_outputs: the outputs returned by the model
    :rtype: dict
    :return: a dictionary that may contain the following keys and values:


        * '**logits**' :  ``List[torch.Tensor]`` or ``torch.Tensor``

          The inputs to the final softmax. Each tensor should have the shape (*batch_size*, *num_labels*) or (*batch_size*, *length*, *num_labels*).

        * '**logits_mask**': ``List[torch.Tensor]`` or ``torch.Tensor``

          0/1 matrix, which masks logits at specified positions. The positions where *mask==0* won't be included in the calculation of loss on logits. Each tensor should have the shape (*batch_size*, *length*).

        * '**labels**': ``List[torch.Tensor]`` or ``torch.Tensor``

          Ground-truth labels of the examples. Each tensor should have the shape (*batch_size*,) or (*batch_size*, *length*).

        .. Note::

            * **logits_mask** only works for logits with shape (*batch_size*, *length*, *num_labels*). It's used to mask along the length dimension, commonly used in sequence labeling tasks.

            * **logits**, **logits_mask** and **labels** should either all be lists of tensors, or all be tensors.

        * '**losses**' :  ``List[torch.Tensor]``

          It stores pre-computed losses, for example, the cross-entropy between logits and ground-truth labels. All the losses stored here would be summed and weighted by `hard_label_weight` and added to the total loss. Each tensor in the list should be a scalar.

        * '**attention**': ``List[torch.Tensor]``

          List of attention matrices, used to compute intermediate feature matching loss. Each tensor should have the shape (*batch_size*, *num_heads*, *length*, *length*) or (*batch_size*, *length*, *length*), depending on what attention loss is used. Details about various loss functions can be found at :ref:`intermediate_losses`.

        * '**hidden**': ``List[torch.Tensor]``

          List of hidden states used to compute intermediate feature matching loss. Each tensor should have the shape (*batch_size*, *length*, *hidden_dim*).

        * '**inputs_mask**' : ``torch.Tensor``

          0/1 matrix, performs masking on **attention** and **hidden**, should have the shape (*batch_size*, *length*).

        .. Note::
          These keys are all optional:

          * If there is no **inputs_mask** or **logits_mask**, then it's considered as no masking.
          * If not there is no intermediate feature matching loss, you can ignore **attention** and **hidden**.
          * If you don't want to add loss of the original hard labels, you can set ``hard_label_weight=0`` in the :class:`~~textbrewer.DistillationConfig` and ignore **losses**.
          * If **logits** is not provided, the KD loss of the logits will be omitted.
          * **labels** is required if and only if  ``probability_shift==True``.
          * You shouldn't ignore all the keys, otherwise the training won't start :)

          In most cases **logits** should be provided, unless you are doing multi-stage training or non-classification tasks, etc.

    Example::

      '''
      Suppose the model outputs are: logits, sequence_output, total_loss
      class MyModel():
        def forward(self, input_ids, attention_mask, labels, ...):
          ...
          return logits, sequence_output, total_loss

      logits: Tensor of shape (batch_size, num_classes)
      sequence_output: List of tensors of (batch_size, length, hidden_dim)
      total_loss: scalar tensor 

      model inputs are:
      input_ids      = batch[0] : input_ids (batch_size, length)
      attention_mask = batch[1] : attention_mask (batch_size, length)
      labels         = batch[2] : labels (batch_size, num_classes)
      '''
      def SimpleAdaptor(batch, model_outputs):
        return {'logits': (model_outputs[0],),
            'hidden': model.outputs[1],
            'inputs_mask': batch[1]}