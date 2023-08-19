# Effective-Kotlin-zh-CN

Effective Kotlin 中文翻译

[在线阅读地址](https://maxzmeng.github.io/Effective-Kotlin-zh-CN/index.html)

#### 当前进度：中文翻译中
- [x] 英文原文搬运
- [ ] 中文翻译
- [ ] 校对




- Part 1 Good Code
    - Chapter 1 Safety
        - [x] 引言
        - [x] 第1条：限制可变性
        - [x] 第2条：最小化变量作用域
        - [x] 第3条：尽快消除平台类型
        - [x] 第4条：不要把推断类型暴露给外部
        - [x] 第5条：在参数与状态上指定你的期望
        - [x] 第6条：尽可能使用标准库中提供的异常
        - [x] 第7条：当不能返回预期结果时，优先使用`null` o或`Failure` 作为返回值
        - [x] 第8条：正确地处理`null`值
        - [x] 第9条：使用`use`关闭资源
        - [x] 第10条：编写单元测试
    - Chapter 2 Readability
        - [x] 引言
        - [x] 第11条：可读性设计
        - [x] 第12条：操作符的含义应与其函数名一致
        - [ ] Item 13 Avoid Returning Or Operating On Unit
        - [ ] Item 14 Specify The Variable Type When It Is Not Clear
        - [ ] Item 15 Consider Referencing Receivers Explicitly
        - [ ] Item 16 Properties Should Represent State Not Behavior
        - [ ] Item 17 Consider Naming Arguments
        - [ ] Item 18 Respect Coding Conventions
- Part 2 Code Design
    - Chapter 3 Reusability
        - [ ] Introduction
        - [ ] Item 19 Do Not Repeat Knowledge
        - [ ] Item 20 Do Not Repeat Common Algorithms
        - [ ] Item 21 Use Property Delegation To Extract Common Property Patterns
        - [ ] Item 22 Use Generics When Implementing Common Algorithms
        - [ ] Item 23 Avoid Shadowing Type Parameters
        - [ ] Item 24 Consider Variance For Generic Types
        - [ ] Item 25 Reuse Between Different Platforms By Extracting Common Modules
    - Chapter 4 Abstraction Design
        - [ ] Introduction
        - [ ] Item 26 Each Function Should Be Written In Terms Of A Single Level Of Abstraction
        - [ ] Item 27 Use Abstraction To Protect Code Against Changes
        - [ ] Item 28 Specify API Stability
        - [ ] Item 29 Consider Wrapping External API
        - [ ] Item 30 Minimize Elements Visibility
        - [ ] Item 31 Define Contract With Documentation
        - [ ] Item 32 Respect Abstraction Contracts
    - Chapter 5 Object Creation
        - [ ] Introduction
        - [ ] Item 33 Consider Factory Functions Instead Of Constructors
        - [ ] Item 34 Consider A Primary Constructor With Named Optional Arguments
        - [ ] Item 35 Consider Defining A DSL For Complex Object Creation
    - Chapter 6 Class Design
        - [ ] Introduction
        - [ ] Item 36 Prefer Composition Over Inheritance
        - [ ] Item 37 Use The Data Modifier To Represent A Bundle Of Data
        - [ ] Item 38 Use Function Types Instead Of Interfaces To Pass Operations And Actions
        - [ ] Item 39 Prefer Class Hierarchies To Tagged Classes
        - [ ] Item 40 Respect The Contract Of Equals
        - [ ] Item 41 Respect The Contract Of Hash Code
        - [ ] Item 42 Respect The Contract Of Compare To
        - [ ] Item 43 Consider Extracting Non Essential Parts Of Your API Into Extensions
        - [ ] Item 44 Avoid Member Extensions
- Part 3 Efficiency
    - Chapter 7 Make It Cheap
        - [ ] Introduction
        - [ ] Item 45 Avoid Unnecessary Object Creation
        - [ ] Item 46 Use Inline Modifier For Functions With Parameters Of Functional Types
        - [ ] Item 47 Consider Using Inline Classes
        - [ ] Item 48 Eliminate Obsolete Object References
    - Chapter 8 Efficient Collection Processing
        - [ ] Introduction
        - [ ] Item 49 Prefer Sequence For Big Collections With More Than One Processing Step
        - [x] 第50条：减少操作的次数
        - [x] 第51条：在“性能优先”的场景，使用基础类型数组
        - [x] 第52条：在处理局部变量时，考虑使用可变集合

