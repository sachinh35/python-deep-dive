# Introduction

This article contains details regarding the internal representation and mechanicals of a `list` data structure in Python.
It assumes that the reader is already familiar with what a `list` is in Python and how it works. The article doesn't 
go over the basics of what a `list`, how it stores elements, and how to access those elements. 

## Internal representation of a list

A `list` actually stores pointers to its elements in contiguous memory blocks, and not the actual elements. Python stores
the value of elements wherever it can find free memory. The elements' memory addresses need not be in contiguous memory
blocks but the pointers to those memory addresses are always stored in contiguous blocks of memory.

Consider the code below:
```python
import ctypes

t = [1, "test", 2.5]

# Address of the list's pointer array (the ob_item field, 24 bytes into the list object)
array_start = ctypes.c_void_p.from_address(id(t) + 24).value

for i in range(len(t)):
    slot_addr = array_start + i * 8                              # where the pointer lives
    points_to = ctypes.c_void_p.from_address(slot_addr).value    # the pointer's value
    element = ctypes.cast(points_to, ctypes.py_object).value     # follow it to the object

    print(f"slot {i} @ {hex(slot_addr)}  ->  {hex(points_to)}  ->  {element!r}  (id = {hex(id(t[i]))})")
```

After you run this code, you should get an output which should be something like the following:
```
Output:

slot 0 @ 0x1048a2510  ->  0x104db6318  ->  1       (id = 0x104db6318)
slot 1 @ 0x1048a2518  ->  0x10432ba50  ->  'test'  (id = 0x10432ba50)
slot 2 @ 0x1048a2520  ->  0x10429b210  ->  2.5     (id = 0x10429b210)
```

In the output above, we can clearly see that slot 0, 1, and 2 each point to a contiguous block of memory addresses, with
increments of 8 bytes. Those pointers then point to the actual memory addresses where the value of the elements in the `list`
is stored. 

We also confirm this in our output with `{hex(id(t[i]))})`. This last piece of code attempts to print the memory address
by directly referencing the element from the list using `t[i]`, like you normally would when working with elements in a list.
We can see that this address is exactly the same as what we get when we are following the memory address pointer with `hex(points_to)`.

Let's dive a little more deep with the questions below.

### 1. Why are the slot addresses stored in increments of 8 bytes?

That's because of working with a 64-bit machine. The "64" in a "64-bit machine" is the size of a memory address in bits:
- 64-bit machine: 64 bits ÷ 8 bits per byte = 8 bytes per memory address
- 32-bit machine: 32 bits ÷ 8 bits per byte = 4 bytes per memory address

### 2. Why do we increment first by 24 bytes to get to `array_start` address?

This question is referring to the line of code below:
```python
array_start = ctypes.c_void_p.from_address(id(t) + 24).value
```
Well, the first 24 bytes are always used for Python's own bookkeeping. Let's take a look at what each of those 3 8-byte 
memory addresses represent:

```python
import ctypes
t = [1, "test", 2.5]
a = id(t)

ctypes.c_ssize_t.from_address(a + 0).value    # 1            -> reference count
ctypes.c_void_p.from_address(a + 8).value     # 0x101819a08  -> same as id(list)
ctypes.c_ssize_t.from_address(a + 16).value   # 3            -> same as len(t)
ctypes.c_void_p.from_address(a + 24).value    # 0x105e16430  -> start of the pointer array
```
- The first 8 bytes store how many other objects are referencing this list. With this, Python can immediately know when
to free the memory assigned to this object when the reference count goes to 0.
- The next 8 bytes store what's the `type` of this object. When you make a Python operation such as `type(t)` Python
will return you `list` as the type of `t`. It knows this because it's stored from bytes 8-15 in the `t` object. We can
confirm this from the following piece of code:

```python
import ctypes

t = [1, 2, 3]

type_pointer = ctypes.c_void_p.from_address(id(t) + 8).value   # read bytes 8-15 of t

print(hex(type_pointer))
print(hex(id(list)))
print(type_pointer == id(list)) # This outputs True
```
- Bytes 8-15 are the same for any object, `int`, `str`, `list`, `tuple`, etc. 
- The next 8 bytes from 16-23 store the length of the list. That's how Python can return `len(t)` in O(1) by directly
referring to the value stored at this memory address. This field is stored for objects having variable length such as
tuples, strings, dictionaries, and so on.
- From byte 24 onwards, it contains the pointers which point to the memory locations which store the 
actual values of elements. 

So that's why we jump ahead 24 bytes before reading the pointers and memory locations.

### Memory usage

TBD