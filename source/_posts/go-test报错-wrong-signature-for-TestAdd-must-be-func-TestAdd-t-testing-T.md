---
title: 'go test报错 wrong signature for TestAdd, must be: func TestAdd(t *testing.T)'
date: 2021-09-21 16:08:40
tags: 
    - Go
---

参考: https://www.cnblogs.com/mingbai/p/gotesterr.html

# 背景

学习 Go 过程中，在终端输入 ``go test`` 显示报错: 

```shell
$ wrong signature for TestAdd, must be: func TestAdd(t *testing.T)
```

后经搜索该报错查明示由于 ``func TestAdd(t testing.T)`` 参数未带 *，需要改为 ``func TestAdd(t *testing.T)``

# 问题复现

报错代码如下: 

```go
// calculation_util_test.go
package calculation_util

import "testing"

func TestAdd(t testing.T) {
	tests := []struct{ a, b, c int }{
		{1, 2, 3},
		{1, 3, 4},
		{1, 1, 2},
	}
	for _, test := range tests {
		actualVal := Add(test.a, test.b)
		expectVal := test.c
		if actualVal != expectVal {
			t.Errorf("Add(%d, %d) actualVal: %d, but expectVal: %d", test.a, test.b, actualVal, expectVal)
		}
	}
}
```

# 如何解决

修改后的代码:

```go
// calculation_util_test.go
package calculation_util

import "testing"

func TestAdd(t *testing.T) {
	tests := []struct{ a, b, c int }{
		{1, 2, 3},
		{1, 3, 4},
		{1, 1, 2},
	}
	for _, test := range tests {
		actualVal := Add(test.a, test.b)
		expectVal := test.c
		if actualVal != expectVal {
			t.Errorf("Add(%d, %d) actualVal: %d, but expectVal: %d", test.a, test.b, actualVal, expectVal)
		}
	}
}
```

测试通过！

