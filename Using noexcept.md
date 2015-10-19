## Using noexcept
 - Posted on June 10, 2011 by Andrzej Krzemieński
 
 
Update. 
This post has been updated based on the input provided by Rani Sharoni, the key author of the noexcept feature, 
in this post on comp.lang.c++.moderated. Previous guidelines for using noexcept were far too limiting. 
I also added a section describing the difference between noexcept and throw(). 
Even if you have already read this article you may want to do it again, as it has changed significantly.
The changes have been highlighted with the blueish color.


Recent discussions in comp.lang.c++.moderated and comp.std.c++ (here are the links) indicate that C++ community
is lacking the information on the new noexcept feature. While we know it is in C++11, there is lots of confusion 
on how it works, what it is good for, why it works the way it does, and how we should use it safely and effectively. 
In this post I try to come up with guidelines on how to use noexcept. They are based on C++ Committee papers I was 
able to collect on the subject, and experts’ replies in discussion groups.

# History

It all started from the observation made by David Abrahams and Douglas Gregor that some types that used to work fine
in C++03 might automatically and silently obtain a throwing move constructor in C++11, and that some operations on STL
containers that used these throwing moves would automatically and silently fail to provide the strong exception safety
guarantee. For detailed description of this issue see article “Exceptionally Moving” in C++Next. The problem has been 
also described in N2855. The document provided the first solution to the problem. A noexcept specifier placed in front
of function’s signature indicating that the function is intended not to throw exceptions:

```c++
noexcept void swap( T& lhs, T& rhs );
```
Compiler would check the function definition and report an error if the function was invoking a potentially throwing 
operation.Move constructors (and destructors) would be implicitly declared as noexcept and therefore compiler would 
statically check if move constructors (and destructors) were really not throwing exceptions. If for some reason you 
needed your move constructor or destructor to be able to throw exceptions, you would have to explicitly declare the 
function as throwing:

```c++
ScopeGuard::~ScopeGuard() throw(...);
```
Such throwing move constructors would not be considered for optimization opportunities in STL containers and thus the
original problem would be solved.

Note that noexcept destructors do not help the original problem of throwing moves. But since the changes in throwing 
detection area were made anyway, it was decided to provide a more general solution that would solve some other long 
outstanding problems: throwing destructors, unimplemented and unsuccessful exception specifications, missing no-throw
optimizations.

Just statically forcing that noexcept functions do not call directly or indirectly any code that might potentially 
throw would be too restrictive. There are functions that never throw but simple static analysis classifies them as 
throwing. Consider the following:

```c++
double sqrt( double x ) {
    if( x < 0 ) throw InvalidInput();
    return nofail_sqrt(x);
}
 
double sqrt_abs( double x ) {
    return sqrt(abs(x));
}
```
Function sqrt_abs never throws because the call to abs eliminates all input to sqrt that could cause a throw.
However simple static analysis says that that sqrt_abs might throw because it calls a function that might throw.
To solve the problem the proposal also offered a noexcept block:

```c++
noexcept double sqrt_abs( double x ) {
    noexcept {
        return sqrt(abs(x));
    }
}
```

Anything inside the block, even if potentially throwing, should not stop the compilation of a noexcept function.
This technically allowed the possibility that a noexcept function could still throw: this would trigger an undefined
behavior. In addition, current exception specifications in C++03 were to be deprecated.

Then, Rani Sharoni proposed a different solution. It was presented at Santa Cruz meeting in 2009: N2983. 
Its main difference in approach was that, rather than banning throwing move operations, it added the ability for STL 
containers to detect whether move operations could throw and if so, resort to using copy operations. This selection 
was performed using the new function template std::move_if_noexcept. While custom move operations are not forced to
be noexcept, an another proposal (N2953) made implicitly generated moves automatically labelled as noexcept wherever
it was possible to verify that they are non-throwing.

Another significant change in the second proposal was that compiler was no longer statically checking if a noexcept
functions really do not throw exceptions. noexcept became more like throw() in C++03: it was just a declaration in 
the function’s interface that you intend your function not to throw, but you could still throw at the risk of
triggering an undefined behavior. While this decision is controversial, it was chosen because:

It had been already implemented on a production-class compiler (Microsoft’s non-standard-conforming implementation
of throw()), and proved successful.The noexcept block for locally disabling the static check was not good enough 
for all situations, for instance it was not suitable for disabling the check in constructors’ initialization lists.
Requiring an additional block in function code was too inconvenient. Additional scope could require rethinking the
scopes of local variables and unnecessarily make the functions longer. Consider qsrt_abs example. We incorrectly 
included function abs into the noexcept block; but if we did not do it, we would have to introduce an automatic 
variable to store the intermediate result, and our one-liner would become a four-liner.
Such static check was too restrictive in case of destructors; lots of current C++03 code would be broken.

As a replacement, N2983 offered a noexcept operator that allows the programmer to statically (at compile-time) 
check if a given expression is a potentially throwing one. Thus, compile-time checks for noexcept violation can
be performed on demand rather than being always forced. Similarly, a couple of type traits, like 
nothrow_move_constructible and nothrow_move_assignable have been added to enable further compile-time querying
for no-throw properties.

Third important thing that N2983 offered was the solution to the often encountered problem of exception 
specifications: how to annotate function templates that may or may not throw exceptions depending on the type(s) 
they are instantiated with. noexcept specification now accepted a boolean parameter that could be computed at 
compile-time indicating if a function may throw or not:

```c++
template< typename T >
void swap( T& lhs, T& rhs )
noexcept( is_nothrow_move_constructible<T>::value && is_nothrow_move_assignable<T>::value )
{
    //...
}
```
Upon next committee’s meeting in Pittsburgh in 2010 the revised version of the proposal, N3050, introduced one important change: the violation of noexcept specification now would result in calling std::terminate and the program would have the freedom to not unwind the stack, or only partially unwind it from the place of an offending throw to the noexcept boundary. This guarantees that the exception will not get out of noexcept function (as it might have been the case with undefined behavior), while still allowing compiler no-throw optimizations. This decision was controversial, and you can see an interesting discussion on the design choices in N3081. One of the main objections was that since throwing from noexcept function has well-defined semantics (calling std::terminate), doing so is perfectly correct, and the compiler cannot reject the following code, even though it can see it is plainly wrong.

```c++
void DoSomethingUseful ();
 
void LaunchApplication() noexcept {
    throw "CallTerminate";
}
 
int main() {
    std::set_terminate( &DoSomethingUseful );
    LaunchApplication();
}
```
This is not as serious as it might sound at first, as the compiler is always allowed to report a warning during the compilation that the noexcept is likely to be violated.

Another important objection is that typically in testing mode library functions that impose some preconditions on their arguments (e.g., std::vector::front requires that the vector is not empty) may wish to validate them and report any violations, and the best way of reporting such violation (although not the only one) is to throw a special exception. Using undefined behavior or implementation-defined behavior in place of std::terminate would allow the compilers to still std::terminate the program in most of the cases while allowing different behavior in special cases. This is well described in N3248.

Third objection was that requiring the program to check for exception specification violation to be able to tell when to call std::terminate would require compiler to insert additional code and make the function run slower compared to the solution with undefined behavior. However it appears to be possible to implement a zero-overhead and still optimized code even with std::terminate requirement when noexcept functions are implemented by calls to only other noexcept functions and language primitives (like ints or pointers) that are known not to throw:

```c++
void fun1() noexcept {
    int i = 0;    // noexcept
    int * p = &i; // noexcept
    p = nullptr;  // noexcept
}
 
void fun2() noexcept {
    fun1();
    fun1();
}
 
int main() {
    std::unique_ptr p( new int(1) );
    fun2();
    // (*) 
}
```

Note that at (*) the compiler does not need to put any exception checking code and can even just put a call to p‘s destructor in the code, rather than registering the destructor for stack unwinding. On the other hand, in function fun2, compiler does not need to add additional code for noexcept violation checks because it knows all the calls inside fun2 are noexcept. Similarly in fun1, no check for noexcept violation is necessary. Thus, in the entire program that is using noexcept functions in such a propagating way, compiler does not need to add any safety-checking code. This is not the case, however, if you call a potentially throwing functions inside noexcept functions, e.g. in sqrt_abs situation, where you cannot statically check the no-throw guarantee. In such cases some violation checking code will be added.

If the noexcept feature appears to you incomplete, prepared in a rush, or in need of improvement, note that all C++ Committee members agree with you. The situation they faced was that a safety problem with throwing move operations was discovered in the last minute and it required a fast solution. The current solution does solve the problem neatly: there will be no silently generated throwing moves that will make STL containers silently break contracts. Apart from that, other usages of noexcept are yet to be thought over and corrected. While the final shape of C++11 is fixed, noexcept is still evolving and will be improved in the next standard versions. The areas of improvement will likely include:

Static checking of no-throw guarantee, with an improved tool for locally disabling the check.
Improved deduction of noexcept specification wherever the compiler is able to do that (see N3227).
Not require the template authors to specify long and unreadable exception specifications in function declarations, including the funny double noexcept syntax (See N3207). The following is a monstrous example that everyone would like to avoid.
```c++
template <class T1, class T2>
struct pair {
    // ...
    constexpr pair() noexcept(
        is_nothrow_constructible<T1>::value &&
        is_nothrow_constructible<T2>::value );
 
    pair(const pair&) = default;
 
    pair(const T1& x, const T2& y) noexcept(  
        is_nothrow_constructible<T1, const T1&>::value &&
        is_nothrow_constructible<T2, const T2&>::value );
    //...
    void swap(pair& p) noexcept( 
        noexcept(swap(first, p.first)) &&
        noexcept(swap(second, p.second)));
};
```
noexcept vs. throw()

One could fairly ask what, in the end, the difference is between noexcept specification and good old throw(). In many aspects they have the same behavior: noexcept operator and ‘nothrow traits’ will equally well detect the no-throw property of the function, regardless if it is declared with noexcept or throw(). As a consequence, the compiler will be able to apply the same no-throw-based optimizations in either variant, for the normal program flow (i.e. as long as no exception violates the no-throw barrier). std::move_if_noexcept will also work equally fine for either.

There are however important differences. In case the no-throw guarantee is violated, noexcept will work faster: it does not need to unwind the stack, and it can stop the unwinding at any moment (e.g., when reaching a catch-all-and-rethrow handler). It will not call std::unexpected. Next, noexcept can be used to express conditional no-throw, like this: noexcept(some-condition)), which is very useful in templates, or to express a may-throw: noexcept(false).

One other non-negligible difference is that noexcept has the potential to become statically checked in the future revisions of C++ standard, whereas throw() is deprecated and may vanish in the future.

Guidelines for using noexcept

Given all the above background we can try to formulate guidelines for when to use noexcept. First and most important one, if you are concerned about reduced performance, or about the risk of calling std::terminate, or you are just not sure about the new feature, or if you have doubts whether you should make your function noexcept or not, then just do not annotate your functions with noexcept at all. The basic idea behind exception mechanism is that practically every function may throw. Your code must be prepared anyway that exceptions get thrown, sometimes even when you may not expect it. Even functions compiled with old C compiler may throw exceptions (because they in turn may call components written in C++). Just let your functions throw.

This does not mean that noexcept is useless. Compiler will annotate implicitly generated member functions of your classes (constructors, copy and move assignments and destructor) with noexcept, as appropriate, and STL components will be querying for this annotation. The feature will enable significant optimizations (by using move constructors) even though you will not see a single noexcept keyword. And when compiler annotates functions with noexcept it does it correctly (except for destructors), so there is no risk that some noexcept move constructor will throw.

It is important to distinguish two situations where functions do not throw exceptions (even in C++03, if you were able to statically inspect the program). First, because it just so happens that you didn’t need to use any throwing or potentially throwing operation to implement the function this time. For an example consider this function.

```c++
int sine( int a, int b ) 
{
    return a / sqrt(a * a  + b * b);
}
```

Integer operations do not throw, square root of a positive number is always well defined (even if it is out of range it will not throw). But the users of sine do not benefit from the fact that it does not throw. They will still be prepared that it might. And you may in the future change the implementation so that it uses an operation that might throw (e.g. on overflow). For this type of functions it is not recommended to annotate them with noexcept because noexcept is in some sense a part of function’s interface. If you commit to it, it will be difficult (if possible) to change it later, when there is a need to throw from the function. On the other hand users of such function (computing sine wave in our example) do not benefit much from the no-throw guarantee.

One additional cost of annotating function as noexcept that needs to be bore in mind is that we are not really declaring that our function does not throw exception: we only declare that we want the program to call std::terminate when we throw. Note that if one day you decide that your function will eventually throw, the compiler might not remind you that you forgot to remove the noexcept specification.

If you insist on using noexcept in those situations, the advice for you is to make your function very small, so that you can easily convince yourself that the function has no way of throwing an exception. You can also use noexcept operator inside your function to statically check if the expressions you use really do not throw exceptions:

```c++
template< typename T >
T sine( T const& a, T const& b ) noexcept
{
    static_assert( noexcept( T(a / sqrt(a * a  + b * b)) ), "throwing expr" );
    return a / sqrt(a * a  + b * b);
}
```
static_assert combined with noexcept operator works almost like statically checked no-throw specification if you carefully check all expressions in your function. However, it may be tricky to check all expressions because some of them are invisible. In the example above, we forgot check if the destructors of types in use may throw exceptions. Type T may not be the only type whose destructor we are calling: decltype(T * T) may not be T. Also, for constructors, the default initialization of sub-objects may go unnoticed.

Another guideline for you, suggested in N3248, is not to annotate your function with noexcept if it is not well defined for all possible input values, i.e. if it imposes some preconditions on its inputs. For instance, the above example does have such precondition: it requires that sqrt(a * a + b * b) != 0. This means that one day you will want to change your function to:

```c++
template< typename T >
T sine( T const& a, T const& b ) noexcept
{
    static_assert( noexcept( T(a / sqrt(a * a  + b * b)) ), "throwing expr" );
    BOOST_ASSERT( sqrt(a * a  + b * b) != 0 );
    return a / sqrt(a * a  + b * b);
}
```
And one day, while unit-testing, you will want to customize the behavior of BOOST_ASSERT so that it throws special exception test_failure. But throwing this exception will then not work because the program is required to std::terminate in such case.

We said there are two situations where functions do not throw exceptions. The second one is when we intend our function to provide a no-fail guarantee or a no-throw guarantee. The difference between the two guarantees is not trivial. A no-fail guarantee indicates that the function will always succeed in doing what it is supposed to. Such functions are needed by other functions to provide a strong (commit-or-rollback) guarantee. A no-throw guarantee is weaker. It indicates that in case the function fails to do its job, it must not throw exceptions: it might use an error code or some other technique for signalling error, or it may just conceal the fact that it failed. This guarantee is required of functions that are called during stack unwinding where some exception is already being handled and throwing a second one makes the unwinding process to fail. These functions include all destructors, constructors of objects representing exceptions, an conversion operators that return exceptions.

For this cases annotating functions with noexcept makes sense. Note that your functions in C++03 already provide no-fail or no-throw guarantees. And the additional keyword doesn’t make the guarantees any stronger. The reason for noexcepting your functions is that in case some programmer that uses them wants, in turn, to provide a no-fail/no-throw guarantee for his function, he will likely want to check your functions with noexcept operator. Also the compiler will be checking for non-throwing specification in order to perform run-time optimizations. Good candidates for such no-fail functions include:

memory deallocation functions,
swap functions,
map::erase and clear functions on containers,
operations on std::unique_ptr,
other operations that the aforementioned functions might need to call.
In case you are not sure if your function is supposed to or might be suspected of providing a no-fail/no-throw guarantee, just assume it is not: do not annotate it with noexcept. You can always safely add the noexcept later, if need be, because you will be only making your guarantees stronger. The inverse is not possible: declaring the function as noexpect is a commitment to your users that they will be relying on.

Next, big performance optimizations can be achieved in containers like STL vector when your type’s move constructor and move assignment are annotated with noexcept. When STL utility std::move_if_noexcept detects that your moves do not throw it will use these safe moves rather than copies for some operations (like resize). This, in case of containers storing millions of elements, will enable huge optimizations. std::move_if_noexcept may not be the only such optimization tool. Hence the next advice: use noexcept specification when you want noexcept operator or one of the following std traits to work:
```c++
is_nothrow_constructible,
is_nothrow_default_constructible,
is_nothrow_move_constructible,
is_nothrow_copy_constructible,
is_nothrow_assignable,
is_nothrow_move_assignable,
is_nothrow_copy_assignable,
is_nothrow_destructible.
```
Very occasionally there may be a need to annotate your function with noexcept(false). For normal functions noexcept(false) specification is implied, but it is not the case for destructors. In C++11 destructors, unless you explicitly specify otherwise, have noexcept(true) specification. There exists a wide consensus in C++ community that destructors shouldn’t throw, however there might exist situations where you still need to be able to throw from destructors. In this case you will need to either annotate your destructor with noexcept(false) or derive your class from another class that already declares a throwing destructor.

The following is a useful tool that uses noexcept specification to implement a no-fail swap function. It builds atop std::swap, which in C++11 is implemented somewhat different:

```c++
template< typename T >
void std::swap( T & a, T & b )
noexcept( is_nothrow_move_constructible<T>::value && is_nothrow_move_assignable<T>::value )
{
    T tmp( move(a) );
    a = move(b);
    b = move(tmp);
}
```
The thing worth noting is that the std::swap is noexcept iff T’s move constructor and move assignment are noexcept. One, but not the only, purpose of function swap is to enable a “modify-the-copy-and-swap” idiom or its special case, “copy-and-swap” idiom, in order to provide a strong exception safety guarantee. In order for this idiom to work we need our swap to provide a no-fail guarantee. Hence the idea of marking swap noexcept wherever possible. On the other hand, for other usages of swap, it does not need to provide no-fail guarantee, and for some types this guarantee is not feasible. Given that, we can use the conditional noexcept flag of std::swap to implement our no-fail version of swapping function:

```c++
template< typename T >
void nofail_swap( T & a, T & b ) noexcept
{
    using std::swap;
    static_assert( noexcept(swap(a, b)) ), "throwing swap" );
    swap(a, b);
}
```
This function should be preferred for implementing “modify-the-copy-and-swap” idiom. Function nofail_swap has the special property: it either does not throw, or the program fails to compile.

Finally, a word of caution for programmers putting moveable but not copyable types, such as std::unique_ptr, or std::thread, into STL containers. Function template std::move_if_noexcept is defined as follows in C++11.

```c++
template< class T > 
typename conditional<
    !is_nothrow_move_constructible<T>::value 
    && is_copy_constructible<T>::value,
        T const&, 
        T &&
>::type move_if_noexcept( T& x ) noexcept;
```
The condition with double negation is somewhat hard to read but it effectively says that we chose to move iff we have a no-throw move or we do not have a copy. That is, if we do not have a copy, we choose to move regardless if the move throws or not. If you use objects of such types in sequence containers std::vector or std::dequeue, note that some operations that typically provide strong exception safety guarantee will now only provide basic guarantee (resize), whereas some other operations may result in unspecified behavior in case any element throws exception while being moved (insert, emplace, emplace_back, push_back, and dequeue-speciffic emplace_front and push_front).
Status API Training Shop Blog About © 2014 GitHub, Inc. Terms Privacy Security Contact 
