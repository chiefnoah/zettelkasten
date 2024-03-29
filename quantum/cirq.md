---
date: 2020-10-21T13:59
tags:
  - quantum-computing
  - work
  - research
  - python
---

# Cirq

## Circuits

Circuits are the primary unit of a #quantum-program, represented using the
`Cirquit` class. A `Cirquit` consists of several `Moment` objects, which itself
is a collection of `Operation` objects. A `Moment` represents a slice of time,
while it's `Operation`s represent some effect on a specific subset of #qubits.

`GateOperation` is the most commonly used type of `Operation`.

`Circuit` is an `iterable` of `Moment` objects.

### Insert Strategies

`InsertStrategy` defines how `Moment`s are automatically constructed when
`Cirquit.append()` is passed multiple `Operation` objects that touch the same
qubit(s). In short, it allows to pass in a series of operations that have a
defined order, but can happen simultaneously so long as the operations are
operating on different qubits. See 
[InsertStrategies](https://cirq.readthedocs.io/en/stable/docs/circuits.html#InsertStrategies).

### OP_TREE

An `OP_TREE` is a contract that states: if the input can be iteratively
flattened into a list of operations, then the input is an `OP_TREE`

### Optimizers

`Circuit`s can be optimized for different hardware architectures or to compact
single qubit gates to optimize runtime complexity/parallelization within
`Moment` time slices. For more details, see 
[Cirq docs](https://cirq.readthedocs.io/en/stable/docs/circuits.html).

### Local R Gates

Implemented as special cases of the `{X,Y,Z}PowGate`s, the rx, ry, and rz gates
are functions that return instances of their corresponding PowGate, converting
the "x-turn" format into radians:
$$
x/\pi
$$

Where $x$ is the `phi` argument to the r{x,y,x} function.

### Serialization

Cirq has some support for JSON serialization. The gates all seem to support the
to_json method, but it looks like `ParallelGateOperation` isn't supported by the
serialization framework.
