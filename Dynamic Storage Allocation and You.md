# How to dynamically allocate memory in C?
There are five and only five functions for dynamic memory management. Memory may be allocated with `malloc()` or `calloc()` or since C11, `aligned_alloc()`. Once allocated, memory must be freed through `realloc()` or `free()`, otherwise there's going to be a memory leak. Sounds easy, right? Let's look at some code:
```c
void foo(int count) {
    int *a = malloc(count * sizeof(int));
    for (int i = 0; i < count; ++i) {
        a[i] = i;
    }
    free(a);
}
```
What could possibly go wrong? Of course, there's no guard statement against invalid inputs, but everyone can see that, what else can you spot?
