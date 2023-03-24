# What is a pointer?
Most sources out there will tell you something like this:

"A pointer is a memory address."

This sounds reasonable, and when you try something like this:

```c
    printf("%p", p);
```

You do get an address, like `0x8D905F1694` or whatever.
But this is only half correct, and a better answer would be:

"A pointer is an address _plus type information_."

Why? There's another common misconception about pointers and memory access. When it come to dereferencing pointers, most sources will give you a graph depicting RAM as a stack of boxes. There's a box labeled `0x8D905F1694`, the next box is labeled `0x8D905F1695`, and so on. Then the process of dereferencing a pointer is depicted as locating the box labeled `0x8D905F1694` and get its contents.

Now, while RAM is indeed divided into "words" that's reasonably similar to boxes, dereferencing is A LOT more than just grabbing the contents of the word the pointer pointed to. Depends on the computer's specifications, the size of a word can vary. A simple `char` usually does not require an whole word, while a struct can potentially require multiple words. Obviously, size information is required so the correct amount of data is retrived from the address.
This is why we have things like `int*` and `char*`, a `int*` basically says: Hey, I have a memory address, I want access it as if it's storing an `int`. And `void*` means: Well, I do have an address, but I don't really know how to access it yet. This is why we have code like this:

```c
    int cmp(const void* a, const void* b) {
        // When we cast pointers, we are basically saying:
        // Hey, I want access this memory as if it's storing [Whatever type you're casting to]
        int *A = (int*) a;
        int *B = (int*) b;
        return (*A > *B) - (*B > *A);
    }
```
With this in mind, double pointers, like `int**` basically means: Hey, here's a memory address, I know there's another memory address located there, and I want to use that memory address as if it's pointing to an `int`. Triple pointers are the same deal: Here's an address, it contains another address, which points to yet another address which finally points to the location I want to access as if it contains [some data type].