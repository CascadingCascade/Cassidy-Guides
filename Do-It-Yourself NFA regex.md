# We are going to build a NFA-based regex matcher, that's it
Theory of computation had always been a favorite subject of mine, (Along side digitial circuit design, physics, philosophy and countless other that is, my interests are quite diverse) there's something simply fascinating about inspecting an automata at work, about how patterns and functions emerge from simple rules. 
But it's a lot less enjoyable to try and implement one yourself, (When you don't really know how do it, that is), for one, directly applying the formal definitions is unlike to helpful. 
But the close association between automatas and graph theory presents opportunities, and in this guide I will walk you through the process of building a very simple NFA-based regex matcher. (And we are doing it in C, even though it's obviously easier with in OOP. I insist that you can never fully learn anything without implementing it in a low level language.)
# How to describe an finite automata?
Formally, a NFA have five components: Starting state(s), recognized alphabets, transition rules, available states, accepting state(s). 
And like I've said, we will not rely too much on this and we are going to do it in the graph theory way. 
Let's begin by defining a transition rule, which would an edge in a graph representation of a NFA:
```c
/**
 * @brief Definition for a NFA state transition rule
 */
struct transRule {
    struct NFAState* target; ///< pointer to the state to transit to
    char cond;               ///< the character required to activate this transition
};
```
There's no information about the state to start from, why? because we don't need to include it.
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
There's no information about which state this state is, because again, we don't really need to.
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
What's up with that `subs` and `wrapperflag`? They will be explained later.
One of the reasons for using pools is that it simplifies memory management, 
when the time comes to destroy an NFA object, we can simply free everything in the pool.
No memory will be leaked because all usages are checked out from the pool.
Using pools also allow us to compare pointers instead of the object they point to.  
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
