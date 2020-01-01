---
title: python function
tags: python
modify_date: 2019-12-23
---

python中的函数。

<!--more-->

# 函数

函数是带名字的代码块，方便代码复用，减少重复代码，方便代码修改和维护。  

## 不带参数的函数

函数由`def`关键字定义，函数名后的括号内指定形参，形参可以为空。  
函数定义中指定的参数称为形参，函数调用时指定的参数称为实参。

```python
def greet():
    """不带参数的函数"""
    
    print("Hello!")
    
greet() #调用函数`greet`
```

## 带参数的函数

函数定义时，可以指定1个或多个形参。

```python
def greet2(username): #带一个形参
    """带参数的函数"""

    print("Hello,", username)

greet2("Jack")#调用函数`greet`时，传入实参"Jack"
```

## 位置实参

位置实参是指：调用函数时，可以按照形参顺序提供实参（即：实参跟形参，按照顺序一一对应）。

```python
def describe_pet(animal_type, pet_name):
    """带2个参数的函数"""

    print("I have a {}.".format(animal_type))
    print("My {}'s name is {}.".format(animal_type, pet_name.title()))

describe_pet("dog", "阿黄")   #调用函数`describe_pet`时，第1个实参`"dog"`会传给第1个形参`animal_type`，第2个实参`"阿黄"`会传给第2个形参`pet_name`。
```

## 关键字实参

关键字实参是指：调用函数时，可以使用名称-值的方式指定实参，python会根据参数名称，将实参与形参对应起来。  
使用关键字实参时，实参位置可以随意。  
**注意：关键字方式指定的实参，必须放到所有位置实参的后边**

```python
def describe_pet(animal_type, pet_name):
    #函数体参考上一节

describe_pet(animal_type="cat", pet_name="小花") #关键字实参方式调用函数时，实参位置可以随意
describe_pet( pet_name="小花", animal_type="cat") #关键字实参方式调用函数时，实参位置可以随意
```

## 默认值

编写函数时，可以给形参指定默认值。  
**注意：指定默认值的形参必须放到形参列表的末尾。**  
调用函数时，就可以不给有默认值的形参指定实参，这时，会自动使用形参的默认值。当然，也可以给有默认值的形参指定实参，python将使用指定的实参。

```python
def describe_pet(pet_name, animal_type="dog"):
    """带默认值的版本"""

    print("I have a {}.".format(animal_type))
    print("My {}'s name is {}.".format(animal_type, pet_name.title())) 

describe_pet(pet_name="阿黄")  #可以不指定第2个实参，则使用第2个形参默认值
describe_pet( "小花", "cat") #可以指定第2个实参，则以指定的实参为准。
```

## 等效的函数调用

由于可以混合使用位置实参、关键字实参和默认值，所以对于一个由多个参数的函数，通常由多种等效的调用方式。

```python
def describe_pet(pet_name, animal_type="dog"):
    #函数体参考上一节

describe_pet(pet_name="阿黄") #使用关键字实参指定第1个参数，省略第2个
describe_pet("小汪") #使用位置实参指定第1个参数，省略第2个
describe_pet("Wallie", "robot") #使用位置实参，指定2个参数
describe_pet("小a", animal_type="cat") #结合位置实参和关键字实参的方式
describe_pet(pet_name="小b", animal_type="cat") #关键字实参指定2个参数
describe_pet(animal_type="cat", pet_name="小c") #关键字实参指定2个参数
```

## 返回值

函数可以使用关键字`return`将指定的值返回到调用函数的代码行。  
函数既可以返回简单类型的值（比如：str，int等），也可以返回复杂类型的值（比如：map，list等）。

```python
def build_person(first_name, last_name, age=None):
    person = {"first_name": first_name, "last_name":last_name}
    if None != age:
        person["age"] = age
    return person

p = build_person("jimi", 'hendrix')
print(p)
```

## 传递列表

当形参是列表时，函数对该列表的修改**会反映到实参上**。  
如果不想作为实参的列表被修改，可以利用切片传入列表的副本。

```python
def eat_food(food_list):
    """把列表中的食物依次吃掉"""

    while food_list:
        food = food_list.pop()
        print("eat food {}".format(food))

foods = ["apple", "rice", "nuddle"]
eat_food(foods)
print(foods) #此时foods已经是空列表了
```

## 传递任意数量的实参

可以在一个形参前加上单个`*`，使该形参可以收集任意数量实参，此时该形参在函数定义中作为元组使用。  
**注意：使用\*修饰的形参可以不是最后一个形参，此时，它之后的普通形参在函数调用时，必须使用关键字实参的方式。**

```python
def say_hi(*persons):
    print(persons)
    for person in persons:
        print("Hi, ", person) 

say_hi("edward", "lily", "david")

def say_hi2(greet_word, *persons):
    print(persons)
    for person in persons:
        print("{}, {}".format(greet_word, person) )

say_hi2("Hello", "edward", "lily", "david")

def say_hi3(greet_word, *persons, leave_word):
    print(persons)
    for person in persons:
        print("{}, {}".format(greet_word, person) )
    print(leave_word)

say_hi3("Hello", "edward", "lily", "david", leave_word="See you!") #定义在*修饰的形参后的形参，在调用时必须使用关键字实参的方式
```

## 传递任意数量的关键字实参

可以在一个形参前加上2个`*`，使该形参可以收集任意数量的名称-值形式的实参，此时该形参在函数定义中作为字典使用。  
**注意：使用2个\*号修饰的形参，必须是最后一个形参。**

```python
def build_profile(first, last, **user_info):
    """创建一个字典，其中包含我们知道的有关用户的一切"""
    
    profile = {}
    profile["first_name"] = first
    profile["last_name"] = last
    for key, value in user_info.items():
        profile[key] = value
    
    return profile

p = build_profile("edward", "Leu", age=20, sex="male")
print(p)
```

## Unpacking Arguments Lists

与“使用一个星号修饰一个形参，使该形参变成能够接纳任意数量的实参的元组”相反，函数调用时，可以使用一个星号修饰一个列表或者元组，从而将它的成员一个一个的解出来，并作为一个一个的实参。  
与“使用2个星号修饰一个形参，使该形参变成能够接纳任意数量的关键字实参的字典”相反，函数调用时可以使用2个星号修饰一个字典，从而将它的成员一个一个的解出来，并作为一个一个的关键字实参。

```python
print(list(range(3, 6)))

args = [3, 6]
print(list(range(*args))) #结果与第1句相同

def build_profile(first, last, **user_info):
    #函数体实现参考上文

p = build_profile("edward", "Leu", age=20, sex="male")
print(p)

args = {"first":"edward", "last":"Leu", "age":20, "sex":"male"}
p = build_profile(**args) #该调用方式，与上边的调用方式等价
print(p)
```

## Special parameters

定义函数时，可以使用`/`和`*`，指定形参只能够使用位置实参方式传入，或者只能够使用关键字实参方式传入，又或者是可以使用2种方式的任一种传入。  
具体的：  
如果，形参列表中有`/`，则`/`之前的形参，必须使用位置实参的方式传入，`/`之后的形参，可以使用2种方式的任一种传入。  
如果，形参列表中没有`/`，则不存在只能够使用位置实参方式传入的形参。  
如果，形参列表中有`*`，则`*`之后的形参，必须使用关键字实参的方式传入，`*`之前的形参，可以使用2种方式的任一种传入。
如果，形参列表中没有`*`，则不存在只能够使用关键字实参方式传入的形参。

```python
def f(pos1, pos2, /, pos_or_kwd, *, kwd1, kwd2):
```

## lambda

## 文档字符串

## Function Annotations

## 编码规范