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
You can learn about this fascinating concept at [Wikipedia](https://en.wikipedia.org/wiki/Abstract_syntax_tree).
To begin building an AST, let's define a node:
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
