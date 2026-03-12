# Basic Programming — Python, JavaScript, and Beyond

Master the fundamentals of programming that every backend engineer needs.

---

## 1. What is Programming?

Programming is giving instructions to a computer in a language it understands.

Think of it like cooking:
- **Recipe** = Program
- **Ingredients** = Data/Variables
- **Steps** = Instructions/Code
- **Final dish** = Output/Result

Every program you write has:
1. **Data** — Information you store (numbers, text, lists)
2. **Logic** — Instructions for what to do with that data
3. **Output** — The result after running the instructions

---

## 2. Core Concepts

### Variables

A variable is a named container that stores data.

**Python:**
```python
name = "Alice"
age = 25
salary = 50000.50
```

**JavaScript:**
```javascript
const name = "Alice";
const age = 25;
const salary = 50000.50;
```

### Data Types

- **String**: Text ("hello", 'world', "123 is not a number")
- **Integer**: Whole numbers (5, -10, 0, 1000)
- **Float**: Decimal numbers (3.14, 2.5, -0.001)
- **Boolean**: True or False
- **List/Array**: Multiple items in one container ([1, 2, 3], ["a", "b", "c"])
- **Dictionary/Object**: Key-value pairs ({"name": "Alice", "age": 25})

### Operators

**Arithmetic:**
```python
5 + 3     # Addition = 8
10 - 4    # Subtraction = 6
3 * 2     # Multiplication = 6
10 / 2    # Division = 5.0
10 % 3    # Modulo (remainder) = 1
2 ** 3    # Exponent = 8
```

**Comparison:**
```python
5 == 5    # Equal? True
5 != 3    # Not equal? True
5 > 3     # Greater than? True
5 < 10    # Less than? True
5 >= 5    # Greater or equal? True
```

**Logical:**
```python
True and False   # Both true? False
True or False    # At least one true? True
not True         # Opposite? False
```

---

## 3. Control Flow

### If / Else

**Python:**
```python
age = 25

if age < 18:
    print("You are a minor")
elif age < 65:
    print("You are working age")
else:
    print("You are retired")
```

**JavaScript:**
```javascript
const age = 25;

if (age < 18) {
    console.log("You are a minor");
} else if (age < 65) {
    console.log("You are working age");
} else {
    console.log("You are retired");
}
```

### Loops

**For Loop (Python):**
```python
for i in range(5):  # 0, 1, 2, 3, 4
    print(i)

for name in ["Alice", "Bob", "Charlie"]:
    print(f"Hello, {name}!")
```

**While Loop (Python):**
```python
count = 0
while count < 5:
    print(count)
    count += 1
```

**For Loop (JavaScript):**
```javascript
for (let i = 0; i < 5; i++) {
    console.log(i);
}

const names = ["Alice", "Bob", "Charlie"];
for (let name of names) {
    console.log(`Hello, ${name}!`);
}
```

---

## 4. Functions

A function is reusable code that does one specific thing.

**Python:**
```python
# Define a function
def greet(name):
    return f"Hello, {name}!"

# Call it
message = greet("Alice")
print(message)  # "Hello, Alice!"
```

**JavaScript:**
```javascript
// Define a function
function greet(name) {
    return `Hello, ${name}!`;
}

// Call it
const message = greet("Alice");
console.log(message);  // "Hello, Alice!"
```

**Function with multiple parameters:**

**Python:**
```python
def add(a, b):
    return a + b

result = add(5, 3)  # 8
```

**JavaScript:**
```javascript
function add(a, b) {
    return a + b;
}

const result = add(5, 3);  // 8
```

---

## 5. Collections (Lists, Dictionaries)

### Lists / Arrays

**Python:**
```python
numbers = [1, 2, 3, 4, 5]
print(numbers[0])      # First element: 1
numbers.append(6)      # Add to end
numbers.remove(3)      # Remove 3
print(len(numbers))    # Length: 5
```

**JavaScript:**
```javascript
const numbers = [1, 2, 3, 4, 5];
console.log(numbers[0]);   // First element: 1
numbers.push(6);           // Add to end
numbers.splice(2, 1);      // Remove at index 2
console.log(numbers.length); // Length: 5
```

### Dictionaries / Objects

**Python:**
```python
user = {
    "name": "Alice",
    "age": 25,
    "city": "New York"
}

print(user["name"])    # "Alice"
user["age"] = 26       # Update
user["email"] = "alice@example.com"  # Add new key
```

**JavaScript:**
```javascript
const user = {
    name: "Alice",
    age: 25,
    city: "New York"
};

console.log(user.name);    // "Alice"
user.age = 26;             // Update
user.email = "alice@example.com";  // Add new key
```

---

## 6. Working with Strings

**Python:**
```python
name = "Alice"

# String concatenation
greeting = "Hello, " + name  # "Hello, Alice"
greeting2 = f"Hello, {name}!"  # f-string (preferred)

# String methods
text = "hello world"
print(text.upper())     # "HELLO WORLD"
print(text.capitalize()) # "Hello world"
print(text.split())     # ["hello", "world"]
```

**JavaScript:**
```javascript
const name = "Alice";

// String concatenation
const greeting = "Hello, " + name;  // "Hello, Alice"
const greeting2 = `Hello, ${name}!`;  // Template literal

// String methods
const text = "hello world";
console.log(text.toUpperCase());  // "HELLO WORLD"
console.log(text.split(" "));     // ["hello", "world"]
```

---

## 7. Error Handling

Errors happen. Good programmers handle them gracefully.

**Python:**
```python
def divide(a, b):
    try:
        result = a / b
        return result
    except ZeroDivisionError:
        print("Cannot divide by zero!")
        return None
    except Exception as e:
        print(f"Error: {e}")
        return None

divide(10, 2)   # 5.0
divide(10, 0)   # "Cannot divide by zero!"
```

**JavaScript:**
```javascript
function divide(a, b) {
    try {
        if (b === 0) {
            throw new Error("Cannot divide by zero!");
        }
        return a / b;
    } catch (error) {
        console.log(`Error: ${error.message}`);
        return null;
    }
}

divide(10, 2);  // 5
divide(10, 0);  // "Error: Cannot divide by zero!"
```

---

## 8. Debugging

When your code doesn't work, you debug.

**Common debugging techniques:**

1. **Print debugging** — Add print statements to see what's happening

```python
def calculate_total(items):
    total = 0
    for item in items:
        print(f"Adding {item}")  # Debug line
        total += item
    print(f"Final total: {total}")  # Debug line
    return total
```

2. **Read error messages** — They tell you what's wrong and where

```
Traceback (most recent call last):
  File "script.py", line 5, in <module>
    print(undefined_variable)
NameError: name 'undefined_variable' is not defined
```

3. **Try simpler code** — Test small pieces independently

```python
# Test just the add function
result = add(5, 3)
print(result)  # Should be 8
```

---

## 9. Hands-On: Your First Programs

### Exercise 1: Calculate Age

**Python:**
```python
birth_year = 1998
current_year = 2026

age = current_year - birth_year
print(f"You are {age} years old")
```

**JavaScript:**
```javascript
const birthYear = 1998;
const currentYear = 2026;

const age = currentYear - birthYear;
console.log(`You are ${age} years old`);
```

### Exercise 2: Check Password Strength

**Python:**
```python
password = "MySecurePass123"

if len(password) < 8:
    print("Password too short")
elif len(password) < 12:
    print("Password is okay")
else:
    print("Password is strong")
```

### Exercise 3: Sum a List

**Python:**
```python
numbers = [10, 20, 30, 40, 50]

total = 0
for num in numbers:
    total += num

print(f"Sum: {total}")  # 150
```

**JavaScript:**
```javascript
const numbers = [10, 20, 30, 40, 50];

let total = 0;
for (let num of numbers) {
    total += num;
}

console.log(`Sum: ${total}`);  // 150
```

---

## 10. Cheat Sheet

| Concept | Python | JavaScript |
|---------|--------|-----------|
| Variable | `x = 5` | `const x = 5;` |
| String | `"hello"` | `"hello"` |
| List | `[1, 2, 3]` | `[1, 2, 3]` |
| Dict | `{"a": 1}` | `{a: 1}` |
| If | `if x > 5:` | `if (x > 5) {` |
| For Loop | `for i in range(5):` | `for (let i = 0; i < 5; i++)` |
| Function | `def func(x):` | `function func(x) {` |
| Return | `return x` | `return x;` |
| Print | `print(x)` | `console.log(x);` |

---

## Next Steps

Now that you understand the basics:

1. **Practice** — Write 5-10 small programs on your own
2. **Explore** — Learn more about your chosen language (Python or JavaScript)
3. **Review** — If anything is unclear, re-read this section
4. **Continue** — Move to [HTTP Requests](02-http-requests.md)

---

## Resources for Further Learning

- **Python**: https://docs.python.org/3/tutorial/
- **JavaScript**: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide
- **Interactive**: https://www.codecademy.com/ (free courses)

Ready to move on? Let's learn about HTTP!

→ **[Next: HTTP Requests](02-http-requests.md)**
