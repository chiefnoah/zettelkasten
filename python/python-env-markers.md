---
date: 2021-03-16T15:51
tags:
  - python
---

# Python Environment Markers

Enviroment markers are used to specify package versions based on the
environment. This can include the platform (ie. `win32` vs `linux`), the python
interpreter version (ie. `2.7` vs `3.5`). They are denoted by a `;` after a
requirement line (in a `requirements.txt`) and followed by the constraint.

For example:

```
requests [security,tests] >= 2.8.1, == 2.8.* ; python_version < "2.7"
```
This allows for only installing packages based on the defined condition.
