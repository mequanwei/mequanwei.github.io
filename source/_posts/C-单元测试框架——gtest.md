---
title: C++单元测试框架——gtest
abbrlink: 2473078899
date: 2019-04-06 13:51:27
tags:
---

最近写C++代码，有时需要测试一两个小函数的功能，需要一个单元测试框架。搜索之后找到 goolge 维护的一个 c++ 单元测试框架——gtest. 
<!--more-->
### 编译安装
在 github [链接](https://github.com/google/googletest)下载代码后，编译即可
``` bash
git clone git@github.com:google/googletest.git
tar zxvf googletest.tar.gz 
cd googletest
mkdir build     
cd build
cmake ../
make 
```
编译完成后，生成如下文件。将其拷贝至自定义位置或者使用 make install 安装至 /usr/local/
```
.
├── include
│   ├── gmock
│   └── gtest
└── lib
    ├── libgmock.a
    ├── libgmock_main.a
    ├── libgtest.a
    |__ libgtest_main.a
```

### 使用

gtest 中定义了很多宏来判断单元测试是否通过。Fatal assertion 和 Nonfatal assertion 的区别在于：Fatal assertion 不通过不会继续执行。Nonfatal assertion 不通过会继续执行后面的测试例。

#### 基础断言
Fatal assertion 	        |Nonfatal assertion	        | Verifies
----                        |----                       |----                |
ASSERT_TRUE(condition)	    |EXPECT_TRUE(condition)	    |condition is true   |
ASSERT_TRUE(condition)	    |EXPECT_TRUE(condition)	    |condition is false  |

#### 二值比较
Fatal assertion 	        |Nonfatal assertion	        | Verifies
----                        |----                       |----                |
ASSERT_EQ(val1, val2)	    |EXPECT_EQ(val1, val2)	    |val1 == val2        |
ASSERT_NE(val1, val2)	    |EXPECT_NE(val1, val2)	    |val1 != val2        |
ASSERT_LT(val1, val2)	    |EXPECT_LT(val1, val2)	    |val1 < val2         |
ASSERT_LE(val1, val2)	    |EXPECT_LE(val1, val2)	    |val1 <= val2        |
ASSERT_GT(val1, val2)	    |EXPECT_GT(val1, val2)	    |val1 > val2         |
ASSERT_GE(val1, val2)	    |EXPECT_GE(val1, val2)	    |val1 >= val2        |

#### 字符串比较
Fatal assertion 	        |Nonfatal assertion	        | Verifies
----                        |----                       |----                                                       |
ASSERT_STREQ(str1,str2) 	|EXPECT_STREQ(str1,str2)	|the two C strings have the same content                    |
ASSERT_STRNE(str1,str2) 	|EXPECT_STRNE(str1,str2)	|the two C strings have different contents                  |
ASSERT_STRCASEEQ(str1,str2) |EXPECT_STRCASEEQ(str1,str2)|the two C strings have the same content, ignoring case     |
ASSERT_STRCASENE(str1,str2) |EXPECT_STRCASENE(str1,str2)|the two C strings have different contents, ignoring case   |

### 创建测试
使用 gtest 创建测试例很简单，使用宏定义格式是：
``` c++
TEST(TestSuiteName, TestName) {
  ... test body ...
}
```
TEST 的第一个参数是 TestSuiteName，第二个参数是 TestName。 
官方例子：
```c++
// Tests factorial of 0.
TEST(FactorialTest, HandlesZeroInput) {
  EXPECT_EQ(Factorial(0), 1);
}

// Tests factorial of positive numbers.
TEST(FactorialTest, HandlesPositiveInput) {
  EXPECT_EQ(Factorial(1), 1);
  EXPECT_EQ(Factorial(2), 2);
  EXPECT_EQ(Factorial(3), 6);
  EXPECT_EQ(Factorial(8), 40320);
}
```
测试函数需要入口，有两种方式设置入口。一是使用传统的 main 函数，二是使用 gtest_main 库。
传统 main 函数方式如下：
``` c++
int main(int argc, char **argv) {
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
```
使用 main 函数作为入口，可以在初始化之前自定义一些测试类，做自定义初始化，使用 TEST_F 等行为。但是， google 的工程师们认为，如果你不需要用这么多复杂的功能，重写 main 函数太麻烦，于是 gtest 有了第二种方法。只需要在链接时链接 `gtest_main` 库，程序会链接至 gtest 自带的 main 函数，用户只需要写测试例即可。

### 编译链接
编译时，链接 pthread库和 gtest 库，gtest_main 库


### 更多
这里只简单的介绍了 gtest 的使用。 gtest 还有更多的扩展功能。参考 github 上官方文档[Googletest Primer](https://github.com/google/googletest/blob/master/googletest/docs/primer.md).
