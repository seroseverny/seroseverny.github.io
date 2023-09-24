---
layout: post
title: Writing a nice geometric transform class in python
---

A common approach for transforming a 3D vector from one coordinate frame to
another involves the following steps:
- representing a coordinate transform as a 4x4 matrix in homogeneous coordinates.
- homogenezing a vector (padding it with a "1")
- matrix multiplication followed by de-homogenization (removing the "1" from the vector).

In C++ `Eigen` provides `Eigen::Isometry3{f,d}` and `Eigen::Affine3{f,d}` that excel at this -- they wrap
a 4x4 matrix, and overload a `*` operator to be applicable to `Vector3{f,d}`
objects. In other words, they abstract away the homogenization/dehomogenization
process -- a user can apply the transform to len-3 vectors without caring
about the details.

As a side note, of course one does not *need* to represent a coordinate
transform by a 4x4 matrix. You can split it into a 3x3 rotation matrix and a 3x1
translation vector, and apply those separately to a len-3 vector. But this is
a short descent into darkness; after two or three transformations, you'll end
up with coordinate transform bugs. Fundamentally, the faux pas occurs when
one splits a single object -- a member of `SE(3)`-- into two objects -- rotation and translation in this case.

Another critic might say that the above transform only performs the correct
thing when applied to points, and not to *vectors* -- which are transformed
by applying a rotation matrix alone. That's a less frequent situation though, and
`Eigen::Isometry3d` class does have a `rotation()` accessor which can be used
for that purpose.

Overall, my position is that use of `Eigen::Isometry3{f,d}` drastically improves readability, and likely protects against bugs.

While in C++ the situation is rosy, in python it isn't, at least not out of the box with numpy.
Therefore, this note tries to design a python `Transform` class that can be
used with numpy for coordinate frame transformations. The gist is to have a
class that wraps a 4x4 matrix and overloads `__mul__` and `__matmul__` operators.

In my opinion, we want the following: ctor to create the object from
any of the feasible representations (4x4 matrix, rotation matrix+translation,
quaternion, or axis-angles), accessors for the full matrix, rotation, and translation
components, and and finally methods for
composing transforms and applying them to points (which together overload
`__mul__` and `__matmul__` operators). An convenience method for the inverse
can also be handy.

This might look like:

```python
class Transform:

  def __init__(self, rotation, translation) -> None:
    # ...deal with various representations...
    self._matrix[0:3, 0:3] = rotation
    self._matrix[0:3, -1] = translation
    pass

  @staticmethod
  def from_matrix(matrix: np.ndarray) -> "Transform":
    return Transform(matrix[0:3, 0:3], matrix[0:3, -1])

  @staticmethod
  def identity() -> "Transform":
    return Transform.from_matrix(np.eye(4))

  def inverse(self) -> "Transform":
    return Transform.from_matrix(np.linalg.inv(self._matrix))

  def compose(self, other: "Transform") -> "Transform":
    return Transform.from_matrix(self._matrix @ other.matrix())

  def apply(self, inputs: np.ndarray) -> np.ndarray:
    pad_width = [(0, 1)] if inputs.ndim == 1 else [(0, 1), (0, 0)]
    inputs = np.pad(inputs, pad_width, mode="constant", constant_values=1)
    return (self._matrix @ inputs)[0:3, ...]

  def matrix(self) -> np.ndarray:
    return self._matrix

  def translation(self) -> np.ndarray:
    return self._matrix[0:3, -1]

  def rotation(self) -> np.ndarray:
    return self._matrix[0:3, 0:3]

  def __mul__(self, other: T) -> T:
    """Have * operator behave like matrix multiplication."""
    if isinstance(other, np.ndarray):
      return self.apply(other)
    elif isinstance(other, Transform):
      return self.compose(other)
    else:
      raise TypeError(f"Cannot combine a Transform object with {type(other)}")

  def __matmul__(self, other: T) -> T:
    """Have @ operator behave like matrix multiplication."""
    return self.__mul__(other)

  def __repr__(self):
    return self._matrix.__repr__()
```

A sketch of the above is outlined in this [gist](https://gist.github.com/seroseverny/7a7a1125c0b149bb2f639241ee4f3efb).
