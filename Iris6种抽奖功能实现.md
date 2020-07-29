# 6种抽奖活动来一遍

## 年会抽奖活动-annualMeeting

- 需求：导入全公司员工名单，每次随机抽取出来一个人

- 未知：有一等奖、二等奖，也有领导临时增加的奖品
- 年会抽奖不涉及并发，但抽奖程序还是要考虑并发安全性问题



  ### 年会抽奖基本功能实现

```go
/**
 * 年会抽奖程序
 * 不是线程安全
 * 基础功能：
 * 1 /import 导入参与名单作为抽奖的用户
 * 2 /lucky 从名单中随机抽取用户
 * 测试方法：
 * curl http://localhost:8080/
 * curl --data "users=yifan,yifan2" http://localhost:8080/import
 * curl http://localhost:8080/lucky
 * @author Anker
 */

package main

import (
	"fmt"
	"math/rand"
	"strings"
	"time"

	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
)

var userList []string

func newApp() *iris.Application {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&lotteryController{})
	return app
}

func main() {
	app := newApp()

	userList = make([]string, 0)

	// http://localhost:8080
	app.Run(iris.Addr(":8080"))
}

// 抽奖的控制器
type lotteryController struct {
	Ctx iris.Context
}

// GET http://localhost:8080/
func (c *lotteryController) Get() string {
	count := len(userList)
	return fmt.Sprintf("当前总共参与抽奖的用户数: %d\n", count)
}

// POST http://localhost:8080/import
func (c *lotteryController) PostImport() string {
	strUsers := c.Ctx.FormValue("users")
	users := strings.Split(strUsers, ",")
	count1 := len(userList)
	for _, u := range users {
		u = strings.TrimSpace(u)
		if len(u) > 0 {
			// 导入用户
			userList = append(userList, u)
		}
	}
	count2 := len(userList)
	return fmt.Sprintf("当前总共参与抽奖的用户数: %d，成功导入用户数: %d\n", count2, (count2 - count1))
}

// GET http://localhost:8080/lucky
func (c *lotteryController) GetLucky() string {
	count := len(userList)
	if count > 1 {
		seed := time.Now().UnixNano()                                // rand内部运算的随机数
		index := rand.New(rand.NewSource(seed)).Int31n(int32(count)) // rand计算得到的随机数
		user := userList[index]                                      // 抽取到一个用户
		userList = append(userList[0:index], userList[index+1:]...)  // 移除这个用户
		return fmt.Sprintf("当前中奖用户: %s, 剩余用户数: %d\n", user, count-1)
	} else if count == 1 {
		user := userList[0]
		userList = userList[0:0]
		return fmt.Sprintf("当前中奖用户: %s, 剩余用户数: %d\n", user, count-1)
	} else {
		return fmt.Sprintf("已经没有参与用户，请先通过 /import 导入用户 \n")
	}

}
```





### 编写web单元测试和并发安全问题

```go
/**
 * 线程是否安全的测试
 * slice切片在并发请求时不会出现异常，但也是线程不安全的，数据更新会有问题
 * go test -v
 */
package main

import (
	"fmt"
	"sync"
	"testing"

	"github.com/kataras/iris/httptest"
)

func TestMVC(t *testing.T) {
	e := httptest.New(t, newApp())

	var wg sync.WaitGroup
	e.GET("/").Expect().Status(httptest.StatusOK).
		Body().Equal("当前总共参与抽奖的用户数: 0\n")

	// 启动100个协程并发来执行用户导入操作
	// 如果是线程安全的时候，预期倒入成功100个用户
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			e.POST("/import").WithFormField("users", fmt.Sprintf("test_u%d", i)).Expect().Status(httptest.StatusOK)
		}(i)
	}

	wg.Wait()

	e.GET("/").Expect().Status(httptest.StatusOK).
		Body().Equal("当前总共参与抽奖的用户数: 100\n")
	e.GET("/lucky").Expect().Status(httptest.StatusOK)
	e.GET("/").Expect().Status(httptest.StatusOK).
		Body().Equal("当前总共参与抽奖的用户数: 99\n")
}
```

### 用互斥锁解决并发安全问题



> golang的多线程固然好用，但是有时候需要对数据进行上锁，防止数据被其它线程更改。那么sync包下的Mutex非常好用。Mutex是一个互斥锁。可以作为struct的一部分，这样这个struct就会防止被多线程更改数据。
> 链接：https://www.jianshu.com/p/e2ff878c1d7b
>
> Go语言互斥锁（sync.Mutex）和读写互斥锁（sync.RWMutex）链接: http://c.biancheng.net/view/107.html

```go
/**
 * 年会抽奖程序
 * 增加了互斥锁，线程安全
 * 基础功能：
 * 1 /import 导入参与名单作为抽奖的用户
 * 2 /lucky 从名单中随机抽取用户
 * 测试方法：
 * curl http://localhost:8080/
 * curl --data "users=yifan,yifan2" http://localhost:8080/import
 * curl http://localhost:8080/lucky
 * @author Anker
 */

package main

import (
	"fmt"
	"math/rand"
	"strings"
	"sync"
	"time"

	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
)

var userList []string
var mu sync.Mutex

func newApp() *iris.Application {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&lotteryController{})
	return app
}

func main() {
	app := newApp()

	userList = make([]string, 0)
	mu = sync.Mutex{}

	// http://localhost:8080
	app.Run(iris.Addr(":8080"))
}

// 抽奖的控制器
type lotteryController struct {
	Ctx iris.Context
}

// GET http://localhost:8080/
func (c *lotteryController) Get() string {
	count := len(userList)
	return fmt.Sprintf("当前总共参与抽奖的用户数: %d\n", count)
}

// POST http://localhost:8080/import
func (c *lotteryController) PostImport() string {
	strUsers := c.Ctx.FormValue("users")
	users := strings.Split(strUsers, ",")
	mu.Lock()
	defer mu.Unlock()
	count1 := len(userList)
	for _, u := range users {
		u = strings.TrimSpace(u)
		if len(u) > 0 {
			// 导入用户
			userList = append(userList, u)
		}
	}
	count2 := len(userList)
	return fmt.Sprintf("当前总共参与抽奖的用户数: %d，成功导入用户数: %d\n", count2, (count2 - count1))
}

// GET http://localhost:8080/lucky
func (c *lotteryController) GetLucky() string {
	mu.Lock()
	defer mu.Unlock()
	count := len(userList)
	if count > 1 {
		seed := time.Now().UnixNano()                                // rand内部运算的随机数
		index := rand.New(rand.NewSource(seed)).Int31n(int32(count)) // rand计算得到的随机数
		user := userList[index]                                      // 抽取到一个用户
		userList = append(userList[0:index], userList[index+1:]...)  // 移除这个用户
		return fmt.Sprintf("当前中奖用户: %s, 剩余用户数: %d\n", user, count-1)
	} else if count == 1 {
		user := userList[0]
		userList = userList[0:0]
		return fmt.Sprintf("当前中奖用户: %s, 剩余用户数: %d\n", user, count-1)
	} else {
		return fmt.Sprintf("已经没有参与用户，请先通过 /import 导入用户 \n")
	}

}
```



## 彩票-ticket

### 彩票实现分析

- 即开即得型，刮刮乐，随机尾号数字
- 定时开奖型，双色球，随机7位数字
- 稳赚不赔，技术难度做谁知道

### 刮刮乐和双色球

```go
/**
 * 彩票
 * 1 即刮即得型（已知中奖规则，随机获取号码来匹配是否中奖）
 * 得到随机数： http://localhost:8080/
 *
 * 2 双色球自选型（从已知可选号码中选择每一个位置的号码，等待开奖结果）
 * 开奖号码： http://localhost:8080/prize
 * 规则参考： https://cp..cn/kj/ssq.html?agent=700007
 */
package main

import (
	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
	"fmt"
	"time"
	"math/rand"
)

func newApp() *iris.Application {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&lotteryController{})
	return app
}

func main() {
	app := newApp()
	// http://localhost:8080
	app.Run(iris.Addr(":8080"))
}

// 抽奖的控制器
type lotteryController struct {
	Ctx iris.Context
}

// 即开即得 GET http://localhost:8080/
func (c *lotteryController) Get() string {
	c.Ctx.Header("Content-Type", "text/html")
	seed := time.Now().UnixNano()					// rand内部运算的随机数
	code := rand.New(rand.NewSource(seed)).Intn(10) 	// rand计算得到的随机数
	var prize string
	switch {
	case code == 1:
		prize = "一等奖"
	case code >=2 && code <= 3:
		prize = "二等奖"
	case code >= 4 && code <= 6:
		prize = "三等奖"
	default:
		return fmt.Sprintf("尾号为1获得一等奖<br/>" +
			"尾号为2或者3获得二等奖<br/>" +
			"尾号为4/5/6获得三等奖<br/>" +
			"code=%d<br/>" +
			"很遗憾，没有获奖", code)
	}
	return fmt.Sprintf("尾号为1获得一等奖<br/>" +
		"尾号2或者3获得二等奖<br/>" +
		"尾号4/5/6获得三等奖<br/>" +
		"code=%d<br/>" +
		"恭喜你获得:%s", code, prize)
}

// 定时开奖 GET http://localhost:8080/prize
func (c *lotteryController) GetPrize() string {
	c.Ctx.Header("Content-Type", "text/html")
	seed := time.Now().UnixNano()
	r := rand.New(rand.NewSource(seed))
	var prize  [7]int
	// 红色球，1-33
	for i:=0; i < 6; i++ {
		prize[i] = r.Intn(33)+1
	}
	// 最后一位的蓝色球，1-16
	prize[6] = r.Intn(16)+1
	return fmt.Sprintf("今日开奖号码是： %v", prize)
}
```





