Executive Summary
-----------------
The ``StochasticDragCoeff`` module applies scalar mean-reverting stochastic noise to aerodynamic drag coefficient by
specializing :ref:`MeanRevertingNoise <meanRevertingNoise>`. It perturbs ``dragCoeff`` in a
``DragGeometryMsgPayload`` and republishes the modified geometry message.

Module Description
------------------
The inherited Ornstein-Uhlenbeck state :math:`x` evolves as

.. math::
   dx = -\frac{1}{\tau}x\,dt + \sqrt{\frac{2}{\tau}}\sigma_{st}\,dW

The output drag coefficient is

.. math::
   C_{D,out} = C_{D,in}(1 + x)

The remaining geometry fields are passed through unchanged.

Message Interfaces
------------------
.. bsk-module-io:: stochasticDragCoeff

   input dragGeomInMsg DragGeometryMsgPayload
      Input drag geometry message containing ``dragCoeff``, projected area, and center-of-pressure data.

   output dragGeomOutMsg DragGeometryMsgPayload
      Output drag geometry message with stochastic correction applied to ``dragCoeff``.

Verification and Testing
------------------------
The module is validated in
``src/simulation/mujocoDynamics/stochasticDragCoeff/_UnitTest/test_stochasticDragCoeff.py``
by checking that the output drag coefficient time series has the expected OU statistics (mean, variance, and
correlation time) for a constant nominal drag geometry input.
