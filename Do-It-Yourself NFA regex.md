# We are going to build a NFA-based regex matcher, that's it.
Theory of computation had always been a favorite subject of mine, (Along side digitial circuit design, physics, philosophy and countless other that is, my interests are quite diverse) there's something simply fascinating about inspecting an automata at work, about how patterns and functions emerge from simple rules. 
But it's a lot less enjoyable to try and build one yourself, (When you don't really know how do it, that is), for one, directly applying the formal definitions is unlike to helpful. 
But the close association between automatas and graph theory does presents opportunities, and in this guide I will walk you through the process of building a very simple NFA-based regex matcher. (And we are doing it in C, even though it's obviously easier with in OOP. I insist that you can never fully learn anything without implementing it in a low level language.)
## How to describe an finite automata?
Formally, a NFA have five components: Starting state(s), recognized alphabets, transition rules, available states, accepting state(s). 
And like I've said, we will not rely too much on this, we are going to do it the graph theory way. 
Let's begin by defining a transition rule, which would be an edge in a graph representation of a NFA:
```c
/**
 * @brief Definition for a NFA state transition rule
 */
struct transRule {
    struct NFAState* target; ///< pointer to the state to transit to
    char cond;               ///< the character required to activate this transition
};
```
There's no information about the state to start from, why? Because we don't need to include it, 
we can simply let the state from which it starts from keep a list of rules starting from this state.
(That's one of the oldest trick in the book for designing stuff: What's really needed is _always a lot less_ than what's appears to be needed. 
_And_ you can never know how big that discrepancy is or what's really necessary and what aren't until you actually get around to building it.)
Let's proceed to defining a state in our automata:
```c
/**
 * @brief Definition for a NFA state. Each NFAState object is initialized
 *        to have a capacity of three rules, since there will only be at most two
 *        outgoing rules, and one empty character circular rule in this algorithm
 */
struct NFAState {
    int ruleCount;            ///< number of transition rules this state have
    struct transRule** rules; ///< the transition rules
};
```
There's no information about which state this state is, because again, we don't really need to, 
its index in the `statePool` will handle that part. 
Take note that we are using an array of pointers, we will use pointer array and pointer comparsions a lot in this guide.
Let's define the NFA itself:
```c
/**
 * @brief Definition for the NFA itself.
 *        statePool[0] is defined to be its starting state,
 *        and statePool[1] is defined to be its accepting state.
 *        for simplicity's sake all NFAs are initialized to have
 *        a small fixed capacity, although due to the recursive nature
 *        of this algorithm this capacity is believed to be sufficient
 */
struct NFA {
    int stateCount;                  ///< the total number of states this NFA have
    struct NFAState** statePool;     ///< the pool of all available states
    int ruleCount;                   ///< the total number of transition rules in this NFA
    struct transRule** rulePool;     ///< the pool of all transition rules
    int CSCount;                     ///< the number of currently active states
    struct NFAState** currentStates; ///< the pool of all active states
    int subCount;                    ///< the number of sub NFAs
    struct NFA** subs;               ///< the pool of all sub NFAs
    int wrapperFlag;                 ///< whether this NFA is a concatenation wrapper
};
```
What's up with that `subs` and `wrapperflag`? The reason for their existence will be explained later.
One of the reasons for using pools is that it simplifies memory management, 
when the time comes to destroy an NFA object, we can simply free everything in the pool.
No memory will be leaked because all usages are checked out from the pool.
Using pools also allow us to compare pointers instead of the object they point to, which is vastly faster and simpler.
(Using pools is actually an established design pattern, in case that you don't know it yet.)
Thus the code to add rules to our NFA reads:
```c
/**
 * @brief add a transition rule to a NFA
 * @param nfa target NFA
 * @param rule the rule to be added
 * @param loc which state this rule will be added to
 * @returns void
 */
void addRule(struct NFA* nfa, struct transRule* rule, int loc) {
    nfa->rulePool[nfa->ruleCount++] = rule;
    struct NFAState* state = nfa->statePool[loc];
    state->rules[state->ruleCount++] = rule;
}
```
Of course, since `statePool[0] is defined to be its starting state, and statePool[1] is defined to be its accepting state.`,
we will need to initialize that when creating a NFA object:
```c
/**
 * @brief creates and initializes a NFA
 * @returns pointer to the newly created NFA
 */
struct NFA* createNFA(void) {
    struct NFA* nfa = malloc(sizeof(struct NFA));

    nfa->stateCount = 0;
    nfa->statePool = malloc(sizeof(struct NFAState*) * 5);
    nfa->ruleCount = 0;
    nfa->rulePool = malloc(sizeof(struct transRule*) * 10);
    nfa->CSCount = 0;
    nfa->currentStates = malloc(sizeof(struct NFAState*) * 5);
    nfa->subCount = 0;
    nfa->subs = malloc(sizeof(struct NFA*) * 5);
    nfa->wrapperFlag = 0;

    addState(nfa, createState());
    addState(nfa, createState());
    return nfa;
}
```
Now our NFA struct looks good enough, let's figure out how to construct one from a given regex input.
## How to build a NFA?
Of course, it's possible to construct NFAs directly from regexs, you can do that too, 
just fuse some of the steps I listed below and you get a program that eats regexs then spits out NFAs without any apparent intermediary.
But since this is supposed to be educative, I am not going to cut any corner. Here's everything we need to do:
1. First, we will perform some preprocessing that turns all implicit concatenations explicit.
2. then, we will construct an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree)(AST) from the preprocessed input.
3. then, we will construct the NFA itself from the AST using the [McNaughton–Yamada–Thompson algorithm](https://en.wikipedia.org/wiki/Thompson%27s_construction).
4. finally, we will perform some postprocessing on the NFA to ensure correct empty character functionalities.
Let's analyse these steps further in their own sections.
### Preprocessing
Normally, all concatenations are implicit, like this:
```
(c|ab*)*(d|ef)*
```
if we use `+` to represent concatenations, it would be:
```
(c|a+b*)*+(d|e+f)*
```
While we won't actually use `+`, the idea of making concatations explicit 
before further processing is obivously quite appealing.
To do that, we simply need to identify where an implicit concatenation is happening, 
then insert a relevant symbol. Here's the code:
```c
/**
 * @brief helper function to determine whether a character should be
 *        considered a character literal
 * @param ch the character to be tested
 * @returns `1` if it is a character literal
 * @returns `0` otherwise
 */
int isLiteral(const char ch) {
    return !(ch == '(' || ch == ')' || ch == '*' || ch == '\n' || ch == '|');
}

/**
 * @brief performs preprocessing on a regex string,
 *        making all implicit concatenations explicit
 * @param input target regex string
 * @returns pointer to the processing result
 */
char* preProcessing(const char* input) {
    const size_t len = strlen(input);
    if(len == 0) {
        char* str = malloc(1);
        str[0] = '\0';
        return str;
    }
    char* str = malloc(len * 2);
    size_t op = 0;

    for (size_t i = 0; i < len - 1; ++i) {
        char c = input[i];
        str[op++] = c;
        // one character lookahead
        char c1 = input[i + 1];

        if( (isLiteral(c) && isLiteral(c1)) ||
            (isLiteral(c) && c1 == '(') ||
            (c == ')' && c1 == '(') ||
            (c == ')' && isLiteral(c1)) ||
            (c == '*' && isLiteral(c1)) ||
            (c == '*' && c1 == '(')
                ) {
            // '\n' is used to represent concatenation
            // in this implementation
            str[op++] = '\n';
        }
    }

    str[op++] = input[len - 1];
    str[op] = '\0';
    return str;
}
```
Take note of that if condition which spanned six lines, 
otherwise this algorithm just does what it says on the tin, there's not much to analyse.
### The Abstract Syntax Tree
When we evaluate a regex by hand, we follow various rules. For example, when we compute this:
```
regex: (c|a*b)*
string: caba
```
We first note that there's star at the regex's end, and the parentheses signified that this star applies to the entire `c|a*b`.
Then we select `c` from `c|a*b`, then `a*b` from `c|a*b` to match that `ab`, repeat `a` once.
But since neither `c` nor `a*b` can match the final `a`, this string does not belongs the regular language described by the regex.
The point to take away from this example is during evaluation, it is vital to have _a sense of precedence and scope_.
Luckily, both concepts can be described simultaneously by the abstract syntax tree. 
An AST constructed from that regex might look like this: 
```
star:
    union:
        literal.
        concatenation:
            star:
                literal.
            literal.
```
You can learn more about this fascinating concept at [Wikipedia](https://en.wikipedia.org/wiki/Abstract_syntax_tree). 
For now we will focus on how to build an AST, let's begin by defining a node:
```c
/**
 * @brief Definition for a binary abstract syntax tree (AST) node
 */
struct ASTNode {
    char content;          ///< the content of this node
    struct ASTNode* left;  ///< left child
    struct ASTNode* right; ///< right child
};
```
Note that since there are only binary and unary operators in our case, a binary tree is sufficient.
To ensure the correct precedence, we must jump from a left parenthesis to its matching right parenthesis while searching, 
ignoring whatever's inside the parentheses temporarily. We will achieve that with this helper function:
```c
/**
 * @brief utility function to locate the first occurrence
 *        of a character in a string while respecting parentheses
 * @param str target string
 * @param key the character to be located
 * @returns the index of its first occurrence, `0` if it could not be found
 */
size_t indexOf(const char* str, char key) {
    int depth = 0;

    for (size_t i = 0; i < strlen(str); ++i) {
        const char c = str[i];

        if(depth == 0 && c == key) {
            return i;
        }
        if(c == '(') depth++;
        if(c == ')') depth--;
    }
    // Due to the way this function is intended to be used,
    // it's safe to assume the character will not appear as
    // the string's first character
    // thus `0` is used as the `not found` value
    return 0;
}
``` 
Here's another utiliy function to make up for C's lack of a builtin equivalent to, say, Java's `substring()`:
```c
/**
 * @brief utility function to create a subString
 * @param str target string
 * @param begin starting index, inclusive
 * @param end ending index, inclusive
 * @returns pointer to the newly created subString
 */
char* subString(const char* str, size_t begin, size_t end) {
    char* res = malloc(end - begin + 2);
    strncpy(res, str + begin, end - begin + 1);
    res[end - begin + 1] = '\0';
    return res;
}
```
With these two utility functions, let's build the AST:
```c
/**
 * @brief recursively constructs a AST from a preprocessed regex string
 * @param input regex
 * @returns pointer to the resulting tree
 */
struct ASTNode* buildAST(const char* input) {

    struct ASTNode* node = createNode('\0');
    node->left = NULL;
    node->right = NULL;
    const size_t len = strlen(input);
    size_t index;

    // Empty input
    if(len == 0) return node;

    // Character literals
    if(len == 1) {
        node->content = input[0];
        return node;
    }

    // Discard parentheses
    if(input[0] == '(' && input[len - 1] == ')') {
        char* temp = subString(input, 1, len - 2);
        destroyNode(node);
        node = buildAST(temp);

        free(temp);
        return node;
    }

    // Union
    index = indexOf(input, '|');
    if(index) {
        node->content = '|';

        char* temp1 = subString(input, 0, index - 1);
        char* temp2 = subString(input, index + 1, len - 1);
        node->left = buildAST(temp1);
        node->right = buildAST(temp2);

        free(temp2);
        free(temp1);
        return node;
    }

    // Concatenation
    index = indexOf(input, '\n');
    if(index) {
        node->content = '\n';

        char* temp1 = subString(input, 0, index - 1);
        char* temp2 = subString(input, index + 1, len - 1);
        node->left = buildAST(temp1);
        node->right = buildAST(temp2);

        free(temp2);
        free(temp1);
        return node;
    }

    // Kleene star
    // Testing with indexOf() is unnecessary here,
    // Since all other possibilities have been exhausted
    node->content = '*';
    char* temp = subString(input, 0, len - 2);
    node->left = buildAST(temp);
    node->right = NULL;

    free(temp);
    return node;
}
```
I trust that you can just read the code and understand it? 
Anyway, the idea is that we go down one level every time a matching pair of parentheses are encountered, 
the function will have as many levels of recursion as the regex inputted.
for the `(c|a*b)*` example earlier, it will be parsed like this:
```
1.
star:
    (c|a*b)
2.
star:
    c|a*b
3.
star:
    union:
        c
        a*b
4.
star:
    union:
        literal.
        concatenation:
            a*
            b
5.
star:
    union:
        literal.
        concatenation:
            star:
                a
            literal.
6.
star:
    union:
        literal.
        concatenation:
            star:
                literal.
            literal.
```
You get the idea? Good.
### Building the NFA from AST
Finally the real stuff, yeah? 
The algorithm we will use is recursive like what we've used to build the AST, 
which is in turn recursive just like the regex itself. 
(Come to think of it, formal language theory really is recursion all the way down.) 
It builds the NFA by combining sub NFAs,
you can read about the details of this algorithm on [Wikipedia](https://en.wikipedia.org/wiki/Thompson%27s_construction), 
here I will just show you the code and analysis where it's warranted:
```c
struct NFA* compileFromAST(struct ASTNode* root) {

    struct NFA* nfa = createNFA();

    // Empty input
    if (root->content == '\0') {
        addRule(nfa, createRule(nfa->statePool[1], '\0'), 0);
        return nfa;
    }

    // Character literals
    if (isLiteral(root->content)) {
        addRule(nfa, createRule(nfa->statePool[1], root->content), 0);
        return nfa;
    }

    switch (root->content) {

        case '\n': {
            struct NFA* ln = compileFromAST(root->left);
            struct NFA* rn = compileFromAST(root->right);

            // Redirects all rules targeting ln's accepting state to
            // target rn's starting state
            redirect(ln, ln->statePool[1], rn->statePool[0]);

            // Manually creates and initializes a special
            // "wrapper" NFA
            destroyNFA(nfa);
            struct NFA* wrapper = malloc(sizeof(struct NFA));
            wrapper->stateCount = 2;
            wrapper->statePool = malloc(sizeof(struct NFAState*) * 2);
            wrapper->subCount = 0;
            wrapper->subs = malloc(sizeof(struct NFA*) * 2);
            wrapper->ruleCount = 0;
            wrapper->rulePool = malloc(sizeof(struct transRule*) * 3);
            wrapper->CSCount = 0;
            wrapper->currentStates = malloc(sizeof(struct NFAState*) * 2);
            wrapper->wrapperFlag = 1;
            wrapper->subs[wrapper->subCount++] = ln;
            wrapper->subs[wrapper->subCount++] = rn;

            // Maps the wrapper NFA's starting and ending states
            // to its sub NFAs
            wrapper->statePool[0] = ln->statePool[0];
            wrapper->statePool[1] = rn->statePool[1];

            return wrapper;
        }
        case '|': {

            struct NFA* ln = compileFromAST(root->left);
            struct NFA* rn = compileFromAST(root->right);
            nfa->subs[nfa->subCount++] = ln;
            nfa->subs[nfa->subCount++] = rn;

            // Adds empty character transition rules
            addRule(nfa, createRule(ln->statePool[0], '\0'), 0);
            addRule(ln, createRule(nfa->statePool[1], '\0'), 1);
            addRule(nfa, createRule(rn->statePool[0], '\0'), 0);
            addRule(rn, createRule(nfa->statePool[1], '\0'), 1);

            return nfa;
        }
        case '*': {
            struct NFA* ln = compileFromAST(root->left);
            nfa->subs[nfa->subCount++] = ln;

            addRule(ln, createRule(ln->statePool[0], '\0'), 1);
            addRule(nfa, createRule(ln->statePool[0], '\0'), 0);
            addRule(ln, createRule(nfa->statePool[1], '\0'), 1);
            addRule(nfa, createRule(nfa->statePool[1], '\0'), 0);

            return nfa;
        }
    }

    // Fallback, shouldn't happen in normal operation
    destroyNFA(nfa);
    return NULL;
}
```
the `redirect()` is defined as:
```c
/**
 * @brief helper function to recursively redirect transition rule targets
 * @param nfa target NFA
 * @param src the state to redirect away from
 * @param dest the state to redirect to
 * @returns void
 */
void redirect(struct NFA* nfa, struct NFAState* src, struct NFAState* dest) {
    for (int i = 0; i < nfa->subCount; ++i) {
        redirect(nfa->subs[i], src, dest);
    }
    for (int i = 0; i < nfa->ruleCount; ++i) {
        struct transRule* rule = nfa->rulePool[i];
        if (rule->target == src) {
            rule->target = dest;
        }
    }
}
```
The algorithm is reasonably straight forward with the exception of concatenation.
It requires that we make the `ln`'s starting state the parent NFA's starting state, 
and `rn`'s accepting state the parent's accepting state, 
as well make `ln`'s accepting state `rn`'s starting state.
To achieve that we first redirect all rules targeting `ln`'s accepting state to 
target `rn`'s starting state, essentially rendering it unreachable. 
(Don't worry, since it's still reachable through `statePool`, no memory is leaked.)
Then we creates a special "wrapper" NFA that doesn't really have any states of its own, 
(Hence the `wrapperFlag` to prevent double freeing when destorying NFA objects.) 
but purely function as a gateway between the two sub NFAs and shallower levels of recursion.
### Postprocessing
Now our NFA is ready 
(or I would like to say, but the title of this section is a dead giveaway, does it not?), 
let's write the transition function:
```c
/**
 * @brief moves a NFA forward
 * @param nfa target NFA
 * @param input the character to be fed into the NFA
 * @returns void
 */
void transit(struct NFA* nfa, char input) {
    struct NFAState** newStates = malloc(sizeof(struct NFAState*) * 10);
    int NSCount = 0;

    if (input == '\0') {
        // In case of empty character input, it's possible for
        // a state to transit to another state that's more than
        // one rule away, we need to take that into account
        for (int i = nfa->CSCount - 1; i > -1; --i) {
            struct NFAState *pState = nfa->currentStates[i];
            nfa->CSCount--;
            struct NFAState** states = malloc(sizeof(struct NFAState*) * 10);
            int sc = 0;
            findEmpty(pState, states, &sc);
            for (int j = 0; j < sc; ++j) {
                if(!contains(newStates,NSCount, states[j])) {
                    newStates[NSCount++] = states[j];
                }
            }
            free(states);
        }
    } else {
        // Iterates through all current states
        for (int i = nfa->CSCount - 1; i > -1; --i) {
            struct NFAState *pState = nfa->currentStates[i];
            // Gradually empties the current states pool, so
            // it can be refilled
            nfa->CSCount--;

            // Iterate through rules of this state
            for (int j = 0; j < pState->ruleCount; ++j) {
                const struct transRule *pRule = pState->rules[j];

                if(pRule->cond == input) {
                    if(!contains(newStates, NSCount, pRule->target)) {
                        newStates[NSCount++] = pRule->target;
                    }
                }

            }
    }
    }

    nfa->CSCount = NSCount;
    for (int i = 0; i < NSCount; ++i) {
        nfa->currentStates[i] = newStates[i];
    }
    free(newStates);
}
```
As for that empty character input special case, kindly examine this image from Wikipedia:
![Example of (ε|a*b) using Thompson's construction, step by step](https://upload.wikimedia.org/wikipedia/commons/thumb/5/52/Small-thompson-example.svg/444px-Small-thompson-example.svg.png)

Since an empty input can be interpreted as an arbitrary numbers of empty characters, 
and an arbitrary number of empty characters can be interpreted as existing between two normal characters, 
the starting state in the image actually should be consider as six states superimposed on each other. 
We need to account for this by adding all states reachable by inputting empty characters to `newStates` pool. 
The `findEmpty()` helper function is define thus:
```c
/**
 * @brief helper function to manage empty character transitions
 * @param target target NFA
 * @param states pointer to results storage location
 * @param sc pointer to results count storage location
 * @returns void
 */
void findEmpty(struct NFAState* target, struct NFAState** states, int *sc) {
    for (int i = 0; i < target->ruleCount; ++i) {
        const struct transRule *pRule = target->rules[i];

        if (pRule->cond == '\0' && !contains(states, *sc, pRule->target)) {
            states[(*sc)++] = pRule->target;
            // the use of `states` and `sc` is necessary
            // to sync data across recursion levels
            findEmpty(pRule->target, states, sc);
        }
    }
}
```
And here's the `contains()` helper function, not much to see here:
```c
/**
 * @brief helper function to determine an element's presence in an array
 * @param states target array
 * @param len length of the target array
 * @param state the element to search for
 * @returns `1` if the element is present, `0` otherwise
 */
int contains(struct NFAState** states, int len, struct NFAState* state) {
    int f = 0;
    for (int i = 0; i < len; ++i) {
        if(states[i] == state) {
            f = 1;
            break;
        }
    }
    return f;
}
```