Basic examples
==============

Here are a few short basic `how to` examples.

How to write and execute a circuit?
-----------------------------------

Here is an example of a circuit with 2 qubits:

.. code-block::  python

    import numpy as np
    from qibo.models import Circuit
    from qibo import gates

    # Construct the circuit
    c = Circuit(2)
    # Add some gates
    c.add(gates.H(0))
    c.add(gates.H(1))
    # Define an initial state (optional - default initial state is |00>)
    initial_state = np.ones(4) / 2.0
    # Execute the circuit and obtain the final state
    final_state = c(initial_state) # c.execute(initial_state) also works
    print(final_state.numpy())
    # should print `np.array([1, 0, 0, 0])`

If you are planning to freeze the circuit and just query for different initial
states then you can use the ``Circuit.compile()`` method which will improve
evaluation performance, e.g.:

.. code-block:: python

    import numpy as np
    # switch backend to "matmuleinsum" or "defaulteinsum"
    # (slower than default "custom" backend)
    import qibo
    qibo.set_backend("matmuleinsum")
    from qibo.models import Circuit
    from qibo import gates

    c = Circuit(2)
    c.add(gates.X(0))
    c.add(gates.X(1))
    c.add(gates.CU1(0, 1, 0.1234))
    c.compile()

    for i in range(100):
        init_state = np.ones(4) / 2.0 + i
        c(init_state)

Note that compiling is only supported when native tensorflow gates are used.
This happens when the calculation backend is switched to ``"matmuleinsum"``
or ``"defaulteinsum"``. This backend is much slower than the default ``"custom"``
backend which uses custom tensorflow operators to apply gates.

How to print a circuit summary?
-------------------------------

It is possible to print a summary of the circuit using ``circuit.summary()``.
This will print basic information about the circuit, including its depth, the
total number of qubits and all gates in order of the number of times they appear.
The QASM name is used as identifier of gates.
For example

.. code-block:: python

    from qibo.models import Circuit
    from qibo import gates

    c = Circuit(3)
    c.add(gates.H(0))
    c.add(gates.H(1))
    c.add(gates.CNOT(0, 2))
    c.add(gates.CNOT(1, 2))
    c.add(gates.H(2))
    c.add(gates.TOFFOLI(0, 1, 2))
    print(c.summary())
    # Prints
    '''
    Circuit depth = 5
    Total number of gates = 6
    Number of qubits = 3
    Most common gates:
    h: 3
    cx: 2
    ccx: 1
    '''

The circuit property ``circuit.gate_types`` will also return a ``collections.Counter``
that contains the gate types and the corresponding numbers of appearance. The
method ``circuit.gates_of_type()`` can be used to access gate objects of specific type.
For example for the circuit of the previous example:

.. code-block:: python

    common_gates = c.gate_types.most_common()
    # returns the list [("h", 3), ("cx", 2), ("ccx", 1)]

    most_common_gate = common_gates[0][0]
    # returns "h"

    all_h_gates = c.gates_of_type("h")
    # returns the list [(0, ref to H(0)), (1, ref to H(1)), (4, ref to H(2))]

A circuit may contain multi-controlled or other gates that are not supported by
OpenQASM. The ``circuit.decompose(*free)`` method decomposes such gates to
others that are supported by OpenQASM. For this decomposition to work the user
has to specify which qubits can be used as free/work. For more information on
this decomposition we refer to the related publication on
`arXiv:9503016 <https://arxiv.org/abs/quant-ph/9503016>`_. Currently only the
decomposition of multi-controlled ``X`` gates is implemented.


.. _measurement-examples:
How to perform measurements?
----------------------------

In order to obtain measurement results from a circuit one has to add measurement
gates (:class:`qibo.base.gates.M`) and provide a number of shots (``nshots``)
when executing the circuit. This will return a :class:`qibo.base.measurements.CircuitResult`
object which contains all the information about the measured samples.
For example

.. code-block:: python

    from qibo.models import Circuit
    from qibo import gates

    c = Circuit(2)
    c.add(gates.X(0))
    # Add a measurement register on both qubits
    c.add(gates.M(0, 1))
    # Execute the circuit with the default initial state |00>.
    result = c(nshots=100)

Measurements are now accessible using the ``samples`` and ``frequencies`` methods
on the ``result`` object. In particular

* ``result.samples(binary=True)`` will return the array ``tf.Tensor([[1, 0], [1, 0], ..., [1, 0]])`` with shape ``(100, 2)``,
* ``result.samples(binary=False)`` will return the array ``tf.Tensor([2, 2, ..., 2])``,
* ``result.frequencies(binary=True)`` will return ``collections.Counter({"10": 100})``,
* ``result.frequencies(binary=False)`` will return ``collections.Counter({2: 100})``.

In addition to the functionality described above, it is possible to collect
measurement results grouped according to registers. The registers are defined
during the addition of measurement gates in the circuit. For example

.. code-block:: python

    from qibo.models import Circuit
    from qibo import gates

    c = Circuit(5)
    c.add(gates.X(0))
    c.add(gates.X(4))
    c.add(gates.M(0, 1, register_name="A"))
    c.add(gates.M(3, 4, register_name="B"))
    result = c(nshots=100)

creates a circuit with five qubits that has two registers: ``A`` consisting of
qubits ``0`` and ``1`` and ``B`` consisting of qubits ``3`` and ``4``. Here
qubit ``2`` remains unmeasured. Measured results can now be accessed as

* ``result.samples(binary=False, registers=True)`` will return a dictionary with the measured sample tensors for each register: ``{"A": tf.Tensor([2, 2, ...]), "B": tf.Tensor([1, 1, ...])}``,
* ``result.frequencies(binary=True, registers=True)`` will return a dictionary with the frequencies for each register: ``{"A": collections.Counter({"10": 100}), "B": collections.Counter({"01": 100})}``.

Setting ``registers=False`` (default option) will ignore the registers and return the
results similarly to the previous example. For example ``result.frequencies(binary=True)``
will return ``collections.Counter({"1001": 100})``.

It is possible to define registers of multiple qubits by either passing
the qubit ids seperately, such as ``gates.M(0, 1, 2, 4)``, or using the ``*``
operator: ``gates.M(*[0, 1, 2, 4])``. The ``*`` operator is useful if qubit
ids are saved in an iterable. For example ``gates.M(*range(5))`` is equivalent
to ``gates.M(0, 1, 2, 3, 4)``.

Unmeasured qubits are ignored by the measurement objects. Also, the
order that qubits appear in the results is defined by the order the user added
the measurements and not the qubit ids.

The measurement gate is only used to sample the measured bitstrings and does not
have  any effect on the state vector. If the user wishes to access the full
state vector of a circuit that was measured, this is possible using the
:py:attr:`qibo.base.circuit.BaseCircuit.final_state` property of circuits.
There reasons that measurement gates do not affect the state are first that
when more than one measurement shots are used, then the final collapsed state
is not uniquely defined and second that the user may wish to resample the final
state vector to obtain more shots, without having to re-execute the simulation.

For applications that require the state vector to be collapsed according to a
single-shot measurement, Qibo provides the :class:`qibo.base.gates.Collapse`
gate. This can be used in any place within the circuit, for example:

.. code-block:: python

    from qibo.models import Circuit
    from qibo import gates

    c = Circuit(2)
    c.add([gates.H(0), gates.H(1)])
    c.add(gates.Collapse(0, result=1))
    final_state = c()
    # ``final_state`` will be [0, 0, 1, 1] / sqrt(2)


Although state collapse typically happens after measurements, currently Qibo
does not allow to add gates to qubits that are measured, and therefore
:class:`qibo.base.gates.Collapse` cannot be used after measurements.
An approach that allows re-using measured qubits after measuring and collapsing
them is to construct a new circuit and pass the old final state as input:

.. code-block:: python

    from qibo.models import Circuit
    from qibo import gates

    c1 = Circuit(5)
    c1.add((gates.H(i) for i in range(5)))
    c1.add(gates.M(0, 2, 3))
    # execute and perform a single shot measurement of qubits 0, 2 and 3
    result = c1(nshots=1)

    # create a new circuit that acts post-measurement
    c2 = Circuit(5)
    # take the measured bitstring from ``result``
    bitstring = result.samples(binary=True)[0]
    # collapse the state according to this bitstring
    c2.add(gates.Collapse(0, 2, 3, result=bitstring))
    # do more gates
    c2.add((gates.H(i) for i in range(5)))
    # execute using the measured final state of ``c1`` as the initial state
    final_state = c2(initial_state=c1.final_state)


How to write a Quantum Fourier Transform?
-----------------------------------------

A simple Quantum Fourier Transform (QFT) example to test your installation:

.. code-block:: python

    from qibo.models import QFT

    # Create a QFT circuit with 15 qubits
    circuit = QFT(15)

    # Simulate final state wavefunction default initial state is |00>
    final_state = circuit()


Please note that the ``QFT()`` function is simply a shorthand for the circuit
construction. For number of qubits higher than 30, the QFT can be distributed to
multiple GPUs using ``QFT(31, accelerators)``. Further details are presented in
the section :ref:`How to select hardware devices? <gpu-examples>`.


.. _precision-example:

How to modify the simulation precision?
---------------------------------------

By default the simulation is performed in ``double`` precision (``complex128``).
We provide the ``qibo.set_precision`` function to modify the default behaviour.
Note that `qibo.set_precision` must be called before allocating circuits:

.. code-block:: python

        import qibo
        qibo.set_precision("single") # enables complex64
        # or
        qibo.set_precision("double") # re-enables complex128

        # ... continue with circuit creation and execution
