# How is memory allocated in C?
Being a quite low level programming language (There is a reason C is sometimes jokingly called "Portable Assembly"), C have some features that's a bit too pedal-to-metal for most people. To properly explain the memory management in C, we need to take a detour to assembly level first(Don't worry, I simplified the hard part for you, my dear reader).
When the `CALL` instruction is executed, the CPU simply pushes the return address onto a stack, the call stack. And when the `RET` is executed, the CPU would pop that address off the call stack and jump to it. Most functions do require arguments, how do we pass them to the function? We can store them in registers, but there's only so many registers, and it's usually desirable to preserve the registers' value before the call.
So the standard practice is to push the arguments to the call stack before the return address. (Even though it's called a stack, the call stack really is just a piece of specially reserved memory, there's nothing against random access.) And since moving a pointer is way faster than sift through the RAM, it becomes another standard practice to store local variables on the call stack. This is why this piece of code will not work except perhaps on the most ridiculous computer in the entire universe:
```c
int main(void) {
    // Not enough space on the stack for that
    int a[100000000];
}
```
and why this piece of code will generate a warning, assuming it didn't stop compilation already:
```c
int* foo(void) {
    int a[5] = {1, 2, 3, 4, 5}; // These values will disappear (become freed) after the function return
    return a; // So the pointer returned points to nothing except garbage values
}
```
Of course, there are many situations where we need more space than what the stack can supply, or we need a variable that can be used beyond the scope it was declared in. This is the where the heap and dynamic memory allocation enter the picture.
In C, there are five and only five functions for dynamic memory management. Memory may be allocated with `malloc()` or `calloc()` or since C11, `aligned_alloc()`. Once allocated, memory must be freed through `realloc()` or `free()`, otherwise there's going to be a memory leak. Sounds easy, right? Let's look at some code:
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
The answer is that there is no check for `malloc()` returning a `NULL`. When allocating large amount of space, `malloc()` and its friends can actually fail, and in that case `a[i] = i;` will generate a segfault due to dereferencing a null pointer. 
Always check before use and after reallocation, and make sure to remember to free all allocated memory in all control paths before returning really is all you need to know. There's a all too common pitfall that I believe a special mention though, is that most people forget to free memory when the function is returning an error. Like this:
```c
int foo(){
    char *str = malloc(100);
    // Do something that can fail here
    if (failed) {
        fprintf(stderr, "Failed!");
        return -1; // No free() here!
    }
    free(str);
    return results;
}
```
