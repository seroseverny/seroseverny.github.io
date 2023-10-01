---
layout: post
title: Standardizing variable names
edit_date: 2023/9/30
original_date: 2021/2/13
---

As the joke goes, naming is one of the hard things<sup>TM</sup> in software engineering.

This note lists some suggestions for variable/method/function names. It's
opinionated and prescriptive.
Most style guides instruct one to choose descriptive variable names. Often they
specifically single out single letter variable names and suggest using
a full word instead, but do not go into further details.
Could there be a set of more concrete advices? This note tries to give some.

The note mainly serves to aggregate several naming rules I followed over the years, and to provide some rationale for them.

### Function prefix/suffix as a documentation of behavior

Most functions can fail (even trivial math operations have to consider
division by zero!). How the function handles failures can be documented via
a suffix in its name.
For example, a function with a "basename" `DoThing()` may be suffixed with
any of the following:

- `basename-OrDie()` (e.g. `DoThingOrDie()`).

    This is a clear sign to the reader that the function does not gracefully
handle errors. Sometimes this arises when `basename()` function exists, and does properly
handle errors, but is inconvenient to use -- i.e. in some contexts error it
is inappropriate to crash, but in others it is preferred to die early.

- `basename-OrNull()` (e.g. `DoThingOrNull()`).

    This one is common in 'getter' functions. It is clear what it returns on failure.
In python, the equivalent is `basename_or_none()`.


Some other, similar patterns are also common:

- `This-And-That` (e.g. `CacheInputsAndSendOutputs()` ).

    If a function performs two distinct actions, it can be documented in the name.

- `Maybe-basename()` (i.e. `MaybeDoThing()` )

    The function may be a no-op depending on the context from which it's called. The
decision as to whether the operation should be done or not is wrapped into the
function body.

### Variable names for pairs/tuples

When dealing with pairs -- and especially containers of pairs -- it is often nice to
use `x_and_y` pattern. For example if you have `Sequence[Tuple[Tensor, Weight]]`
you might choose `tensors_and_weights` as the variable name. If you follow
this pattern, tuple unpacking looks very natural -- basically you never have to
remember which of the two elements is which. For example, in python you
might see things like:

```python
for tensor, weight in tensors_and_weights:
  do_something(tensor, weight)
```

while in C++ you might see something like:

```cpp
  DoSomething(std::get<0>(tensor_and_weight), std::get<1>(tensor_and_weight));
```

When the pattern is used consistently, the reader will immediately recognize
the type (a pair) from a variable name.
This probably does not generalize to 3-tuples and larger, but those are
not frequently used.

### Names for maps, dicts, and other associative containers

The pattern is similar to the one used for pairs, but instead of using `and` to
"join" first and second elements, one uses `by` or `to` to do the same.
As a general rule, given a `Mapping[KeyType, ValueType]`, the corresponding variable
name could be `value_by_key` or `key_to_value`. As a specific example, a map from a
tensor name (a string) to a tensor (an object analogous to `tf.Tensor`) may be
represented by a variable `name_to_tensors` or `tensors_by_name`.

Similar to the above, when this pattern is used consistently, suffixes `_to_`
and `_by_` will immediately be recognized with maps, even in absence of
explicitly annotated types.

Another common approach is to add `_map` suffix to associate containers (e.g.
`tensor_map`). Even this is superior to a variable name like `tensors` -- which
simply states that the object represent a collection.

<i>This is a point of debate!</i> Not everyone appreciates use of container types as a variable name suffix.

### Optionals and StatusOr's

An object of type `std::optional<T>` should be named with a `maybe_x`
prefix. The same suffix can be used for a pointer that could be null.
An object of type `absl::StatusOr<T>` should be named with a `x_or` suffix.
An object of type `absl::Status` should be named with a `_status` suffix
(or simply a `status`).

The reasonably common `absl::` types are used only as an example; the
suggestion of course generalizes to all other optional/result types.

<i>This is a point of debate!</i> Many people dislike this pattern.

Inconsistently with the above (and only applicable to C++), I dislike using
`x_ptr` suffix for pointer types, unless getting to the object requires
multiple dereferencing steps.

A python function/method that can return a `None` could be named with a
`x_or_none` suffix. As an example:
```python
def transform_to_tuple_or_none(x):
  return _transform_to_tuple(x) if _inputs_satisfy_precondition(x) else None
```

### Verbs for bools

One should use verbs to name booleans. Here is an example motivating this
general rule. Below, the meaning of `filter` is unclear without reading the
docstring or the code itself. Is it a callable passed into the function, an
int (representing filter size), a bool?

<div class="code-bad">
{% highlight python %}
def transform_points_into_homogeneous_coords(inputs, filter):
  # function body ...
{% endhighlight %}
</div>

If one used a variable name like `should_filter` or `apply_filter`, the
uncertainty would be eliminated: answers to either of these questions are "yes" or "no", so
it's easier to infer that the variable represents a bool.

The claim is then that:
```python
def transform_points_into_homogeneous_coords(inputs, should_filter):
  # function body
```
reads better.

In particular, it seems to read better because whereas "filter" can be interpreted
as a noun or as a verb (an action), "should\_filter" can only be interpreted
as a verb.

Yes, type hints may help, but only when looking at the function definition, not
when reading the callsite.

A related situation arises when bools are used as a "once" flag -- to record
that an action was done. In this case, it's common to use `did_xxx` pattern,
where `xxx` refers to the thing that is done only once. Something like:

```cpp
void SomeClass::MaybeWarmUpCache() {
  if (did_warm_up_cache) {
    return;
  }
  // Create and warp up cache.
  lru_ = std::make_unique<LruCache>(/*capacity=*/10);
  PopulateCacheDefaultElements(*lru_);
  did_warm_up_cache_ = true;
}
```

### Filename naming

Use `xxx_filename` suffix (e.g. `output_filename`) for strings that store
a path/filename. Do not use `file` -- it is reserved for the actual file descriptor
object. Do not use `fn` -- it is reserved for callables.

### Callable/functor naming

Use `xxx_fn` or `xxx_func` as a suffix for variables that represent callables/functors.
For example:

```cpp
auto resize_func = CreateResizeFunctor(/*output_height=*/100, /*output_width=*/200);
cv::Mat image_1_resized = resize_func(image_1);
cv::Mat image_2_resized = resize_func(image_2);
```

### Filtering

Avoid the use of "filter"; instead use verbs like "discard"/"keep".

Data processing often involves situations where a subset of the data needs to
be discarded or "filtered" according to some condition. As an example, consider
the signature below. The method applies `filter_func` to each example in the
dataset and returns a filtered dataset.

<div class="code-bad">
{% highlight cpp %}
Dataset& Dataset::Filter(std::function<bool(const Image&)>& filter_func);

auto filter_func = [](const Image& x) { return x.rows() > 100; };
dataset = dataset.Filter(filter_func);
{% endhighlight %}
</div>

This is ambiguous though -- does the resulting dataset contain large images
or small images? When `filter_func` returns `true` is the element kept or
discarded? This ambiguity results because the term "filtering" is used so
frequently in so many different contexts.

For this reason, it seems better to try to avoid the term altogether, and
replace it with something like:

{% highlight cpp %}
Dataset& Dataset::EraseIf(std::function<bool(const Image&)>& should_discard_func);
Dataset& Dataset::KeepIf(std::function<bool(const Image&)>& should_keep_func);

auto should_keep_func = [](const Image& x) { return x.rows() > 100; };
dataset = dataset.KeepIf(should_keep_func);
{% endhighlight %}

### Geometric transformations

This section deals specifically with geometric transformations and
coordinate frames. These are so error-prone that naming conventions are essentially
mandatory.
Always use `y_from_x` when naming a geometric transformation that describes a
change of coordinate frame `x` into a coordinate frame `y`. This convention has
a nice property in that inner frames are eliminated in chained expressions,
i.e. `z_from_x = z_from_y * y_from_x`.

If you have an object in one coordinate frame and it is transformed into
another frame, specify this in the variable name, as `object_in_x`.
As a concrete example:

```cpp
Eigen::Vector2d point_in_image(10, 10);
Eigen::Vector3d point_in_camera = camera.Unproject(point_in_image);
Eigen::Vector3d point_in_world = world_from_camera * point_in_camera;
```

This also works nicely for function arguments -- if you require input to be
in a particular coordinate frame, embed this requirement into the variable name.

While working on 2D/3D geometry problems, it is common to transform points from
2D coordinate frame (an image plane) into 3D and vice versa. One pattern that
I've seen for naming such functions is:
- 3D->2D: `Project()`
- 2D->3D: `Unproject()` or `Raycast()`
Note that this convention is different from common English, where the word
"projection" can mean either the 3D->2D transformation or the opposite.

### On auto

Do not use `auto` when applying dereferencing multiple times in a single expression: tell the reader what the type is.

Auto is super useful: it allows us to "hide" unimportant implementation types (remember writing out iterator types in C++03?). But if the object type is not insane, and where the object is accessed through several levels of accessors, I think its use harms readability. Although this comment focuses on C++, it can also be applied to python -- one can annotate the type during the assignment.

This is not a hard rule, but here is an attempt to make the above statement more precise.

Wrapper objects are very common in C++. For example, given a class
`Thing`, in some cases you might need to write a `ThingHolder` or a `ThingWrapper`,
that provides an accessor to the `Thing` itself. Things like containers (e.g. `std::vector`)
or smart pointers are other instances of 'wrapper' objects.
As a concrete example, suppose you have:
```cpp
class ThingHolder {
  Thing& get() { return thing_; }
  /* other methods */
private:
  Thing thing_;
};
```
and have a function like:

<div class="code-bad">
{% highlight cpp %}
void DoSomething(std::vector<std::shared_ptr<ThingHolder>>& wrapped_things) {
  /* some operations here */
  auto& thing = wrapped_things[idx]->get();
  thing.Process(/*..*/);
}
{% endhighlight %}
</div>

In my opinion, this isn't a good place for `auto` -- the type should be explicit instead:
```cpp
  Thing& thing = wrapped_things[idx]->get();
```
We presumably are actually interested in operating on the `Thing` object.
The fact that it's wrapped in a vector of pointers or `ThingHolder`'s
is an implementation detail. Moreover, to get it we have to perform three
operations (access a vector element, deference a pointer, and call `get() `method).
That's quite a lot for a single line.

### Quirks and atypical behavior

Sometimes it's necessary to write code that does something quirky -- not
exactly what one would expect. If this behavior is gated by an option in a
config, consider giving it an unusually long and descriptive name. Long names
stand out when code is formatted, which acts as a visual sign to readers that
something weird<sup>TM</sup> may be going on. The same is true for CLI users who need
to type a very long option -- this should be a sign that the command has caveats
or that it may be dangerous.
As a really benign example -- `git diff` has an option named `--find-copies`
but also `find-copies-harder` (the latter is more thorough, but slower than
the former).

### Single variable names

Yeah: there is nothing wrong with those, especially if you are doing math.

### Related:

 - [a small guide for naming stuff in front-end code](https://blog.frankmtaylor.com/2021/10/21/a-small-guide-for-naming-stuff-in-front-end-code/)


