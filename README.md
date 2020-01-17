# CPP17-StricterExpressionEvaluationOrder
Stricter Expression Evaluation Order

Until C++17 the language hasn’t specified any evaluation order for function parameters.
Period.
For example, that’s why in C++14 make_unique is not just syntactic sugar, but it guarantees
memory safety:
Consider the following examples:
foo(unique_ptr<T>(new T), otherFunction()); // first case
And with explicit new.
foo(make_unique<T>(), otherFunction()); // second case
Considering the first case, in C++14, we only know that new T is guaranteed to happen
before the unique_ptr construction, but that’s all. For example, new T might be called
first, then otherFunction(), and then the constructor unique_ptr is invoked.
For such evaluation order, when otherFunction() throws, then new T generates a leak
(as the unique pointer is not yet created).
When you use make_unique, as in the second case, the leak is not possible as you wrap
memory allocation and creation of unique pointer in one call.
C++17 addresses the issue shown in the first case. Now, the evaluation order of function arguments
is “practical” and predictable. In our example, the compiler won’t be allowed to call
otherFunction() before the expression unique_ptr<T>(new T) is fully evaluated.
  
The Changes
In an expression:
f(a, b, c);
The order of evaluation of a, b, c is still unspecified, but any parameter is fully evaluated
before the next one is started. It’s especially crucial for complex expressions like this:

f(a(x), b, c(y));
If the compiler chooses to evaluate a(x) first, then it must evaluate x before processing b,
c(y) or y.
This guarantee fixes the problem with make_unique vs unique_ptr<T>(new T()). A
given function argument must be fully evaluated before other arguments are evaluated.
  
  #include <iostream>
class Query {
public:
Query& addInt(int i) {
std::cout << "addInt: " << i << '\n';
return *this;
}
Query& addFloat(float f) {
std::cout << "addFloat: " << f << '\n';
return *this;
}
};
float computeFloat() {
std::cout << "computing float... \n";
return 10.1f;
}
float computeInt() {
std::cout << "computing int... \n";
return 8;
}
int main() {
Query q;
q.addFloat(computeFloat()).addInt(computeInt());
}
  
You probably expect that using C++14 computeInt() happens after addFloat. Unfortunately,
that might not be the case. For instance here’s an output from GCC 4.7.3:

computing float...
addFloat: 10.1
addInt: 8
The chaining of functions is already specified to work from left to right (thus addInt()
happens after addFloat()), but the order of evaluation of the inner expressions can differ.
To be precise:
The expressions are indeterminately sequenced with respect to each other.
With C++17, function chaining will work as expected when they contain inner expressions,
i.e., they are evaluated from left to right:
In the expression:
a(expA).b(expB).c(expC)

expA is evaluated before calling b().
Compiling the previous example with a conformant C++17 compiler, yields the following
result:
computing float...
addFloat: 10.1
computing int...
addInt: 8
Another result of this change is that when using operator overloading, the order of evaluation
is determined by the order associated with the corresponding built-in operator.
For example:
std::cout << a() << b() << c();
The above code contains operator overloading and expands to the following function
notation:

operator<<(operator<<(operator<<(std::cout, a()), b()), c());
Before C++17, a(), b() and c() could be evaluated in any order. Now, in C++17, a() will
be evaluated first, then b() and then c().
Here are more rules described in the paper P0145R3¹:
the following expressions are evaluated in the order a, then b:
1. a.b
2. a->b
3. a->*b
4. a(b1, b2, b3) // b1, b2, b3 - in any order
5. b @= a // '@' means any operator
6. a[b]
7. a << b
8. a >> b
If you’re not sure how your code might be evaluated, then it’s better to make it simple and
split it into several clear statements. You can find some guides in the Core C++ Guidelines,
for example ES.44² and ES.44³.
