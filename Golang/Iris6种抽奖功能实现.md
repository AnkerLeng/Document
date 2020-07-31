# 使用Iris实现6种抽奖活动

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

## 微信摇一摇-wechatShake

```go
/**
 * 微信摇一摇
 *
 * 基础功能：
 * /lucky 只有一个抽奖的接口，奖品信息都是预先配置好的
 * 测试方法：
 * curl http://localhost:8080/
 * curl http://localhost:8080/lucky
 * 压力测试：（线程不安全的时候，总的中奖纪录会超过总的奖品数）
 * wrk -t10 -c10 -d5 http://localhost:8080/lucky
 */

package main

import (
	"fmt"
	"log"
	"math/rand"
	"os"
	"time"

	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
)

// 奖品类型，枚举 iota 从0开始自增
const (
	giftTypeCoin      = iota // 虚拟币
	giftTypeCoupon           // 优惠券，不相同的编码
	giftTypeCouponFix        // 优惠券，相同的编码
	giftTypeRealSmall        // 实物小奖
	giftTypeRealLarge        // 实物大奖
)

// 最大号码
const rateMax = 10000

// 奖品信息
type gift struct {
	id       int      // 奖品ID
	name     string   // 奖品名称
	pic      string   // 照片链接
	link     string   // 链接
	gtype    int      // 奖品类型
	data     string   // 奖品的数据（特定的配置信息，如：虚拟币面值，固定优惠券的编码）
	datalist []string // 奖品数据集合（特定的配置信息，如：不同的优惠券的编码）
	total    int      // 总数，0 不限量
	left     int      // 剩余数
	inuse    bool     // 是否使用中
	rate     int      // 中奖概率，万分之N,0-10000
	rateMin  int      // 大于等于，中奖的最小号码,0-10000
	rateMax  int      // 小于，中奖的最大号码,0-10000
}

// 文件日志
var logger *log.Logger

// 奖品列表
var giftlist []*gift

func main() {
	app := newApp()

	// http://localhost:8080
	app.Run(iris.Addr(":8080"))
}

// 初始化奖品列表信息（管理后台来维护）
func initGift() {
	giftlist = make([]*gift, 5)
	// 1 实物大奖
	g1 := gift{
		id:      1,
		name:    "手机N7",
		pic:     "",
		link:    "",
		gtype:   giftTypeRealLarge,
		data:    "",
		total:   1000,
		left:    1000,
		inuse:   true,
		rate:    10000,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[0] = &g1
	// 2 实物小奖
	g2 := gift{
		id:      2,
		name:    "安全充电 黑色",
		pic:     "",
		link:    "",
		gtype:   giftTypeRealSmall,
		data:    "",
		total:   5,
		left:    5,
		inuse:   true,
		rate:    100,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[1] = &g2
	// 3 虚拟券，相同的编码
	g3 := gift{
		id:      3,
		name:    "商城满2000元减50元优惠券",
		pic:     "",
		link:    "",
		gtype:   giftTypeCouponFix,
		data:    "mall-coupon-2018",
		total:   5,
		left:    5,
		rate:    5000,
		inuse:   true,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[2] = &g3
	// 4 虚拟券，不相同的编码
	g4 := gift{
		id:       4,
		name:     "商城无门槛直降50元优惠券",
		pic:      "",
		link:     "",
		gtype:    giftTypeCoupon,
		data:     "",
		datalist: []string{"c01", "c02", "c03", "c04", "c05"},
		total:    50,
		left:     50,
		inuse:    true,
		rate:     2000,
		rateMin:  0,
		rateMax:  0,
	}
	giftlist[3] = &g4
	// 5 虚拟币
	g5 := gift{
		id:      5,
		name:    "社区10个金币",
		pic:     "",
		link:    "",
		gtype:   giftTypeCoin,
		data:    "10",
		total:   5,
		left:    5,
		inuse:   true,
		rate:    5000,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[4] = &g5

	// 整理奖品数据，把rateMin,rateMax根据rate进行编排
	rateStart := 0
	for _, data := range giftlist {
		if !data.inuse {
			continue
		}
		data.rateMin = rateStart
		data.rateMax = data.rateMin + data.rate
		if data.rateMax >= rateMax {
			// 号码达到最大值，分配的范围重头再来
			data.rateMax = rateMax
			rateStart = 0
		} else {
			rateStart += data.rate
		}
	}
	fmt.Printf("giftlist=%v\n", giftlist)
}

// 初始化日志信息
func initLog() {
	f, _ := os.Create("/var/log/lottery_demo.log")
	logger = log.New(f, "", log.Ldate|log.Lmicroseconds)
}

func newApp() *iris.Application {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&lotteryController{})
	// 初始化日志信息
	initLog()
	// 初始化奖品信息
	initGift()
	return app
}

// 抽奖的控制器
type lotteryController struct {
	Ctx iris.Context
}

// GET http://localhost:8080/
func (c *lotteryController) Get() string {
	count := 0
	total := 0
	for _, data := range giftlist {
		if data.inuse && (data.total == 0 ||
			(data.total > 0 && data.left > 0)) {
			count++
			total += data.left
		}
	}
	return fmt.Sprintf("当前有效奖品种类数量: %d，限量奖品总数量=%d\n", count, total)
}

// GET http://localhost:8080/lucky
func (c *lotteryController) GetLucky() map[string]interface{} {
	code := luckyCode()
	ok := false
	result := make(map[string]interface{})
	result["success"] = ok
	for _, data := range giftlist {
		if !data.inuse || (data.total > 0 && data.left <= 0) {
			continue
		}
		if data.rateMin <= int(code) && data.rateMax > int(code) {
			// 中奖了，抽奖编码在奖品中奖编码范围内
			sendData := ""
			switch data.gtype {
			case giftTypeCoin:
				ok, sendData = sendCoin(data)
			case giftTypeCoupon:
				ok, sendData = sendCoupon(data)
			case giftTypeCouponFix:
				ok, sendData = sendCouponFix(data)
			case giftTypeRealSmall:
				ok, sendData = sendRealSmall(data)
			case giftTypeRealLarge:
				ok, sendData = sendRealLarge(data)
			}
			if ok {
				// 中奖后，成功得到奖品（发奖成功）
				// 生成中奖纪录
				saveLuckyData(code, data.id, data.name, data.link, sendData, data.left)
				result["success"] = ok
				result["id"] = data.id
				result["name"] = data.name
				result["link"] = data.link
				result["data"] = sendData
				break
			}
		}
	}

	return result
}

// 抽奖编码
func luckyCode() int32 {
	seed := time.Now().UnixNano()                                 // rand内部运算的随机数
	code := rand.New(rand.NewSource(seed)).Int31n(int32(rateMax)) // rand计算得到的随机数
	return code
}

// 发奖，虚拟币
func sendCoin(data *gift) (bool, string) {
	if data.total == 0 {
		// 数量无限
		return true, data.data
	} else if data.left > 0 {
		// 还有剩余
		data.left = data.left - 1
		return true, data.data
	} else {
		return false, "奖品已发完"
	}
}

// 发奖，优惠券（不同值）
func sendCoupon(data *gift) (bool, string) {
	if data.left > 0 {
		// 还有剩余的奖品
		left := data.left - 1
		data.left = left
		return true, data.datalist[left]
	} else {
		return false, "奖品已发完"
	}
}

// 发奖，优惠券（固定值）
func sendCouponFix(data *gift) (bool, string) {
	if data.total == 0 {
		// 数量无限
		return true, data.data
	} else if data.left > 0 {
		data.left = data.left - 1
		return true, data.data
	} else {
		return false, "奖品已发完"
	}
}

// 发奖，实物小
func sendRealSmall(data *gift) (bool, string) {
	if data.total == 0 {
		// 数量无限
		return true, data.data
	} else if data.left > 0 {
		data.left = data.left - 1
		return true, data.data
	} else {
		return false, "奖品已发完"
	}
}

// 发奖，实物大
func sendRealLarge(data *gift) (bool, string) {
	if data.total == 0 {
		// 数量无限
		return true, data.data
	} else if data.left > 0 {
		data.left--
		return true, data.data
	} else {
		return false, "奖品已发完"
	}
}

// 记录用户的获奖记录
func saveLuckyData(code int32, id int, name, link, sendData string, left int) {
	logger.Printf("lucky, code=%d, gift=%d, name=%s, link=%s, data=%s, left=%d ", code, id, name, link, sendData, left)
}
```

### 并发安全写法

```go
/**
 * 微信摇一摇
 * 增加互斥锁，保证并发更新数据的安全
 * 基础功能：
 * /lucky 只有一个抽奖的接口，奖品信息都是预先配置好的
 * 测试方法：
 * curl http://localhost:8080/
 * curl http://localhost:8080/lucky
 * 压力测试：（线程不安全的时候，总的中奖纪录会超过总的奖品数）
 * wrk -t10 -c10 -d5 http://localhost:8080/lucky
 */

package main

import (
	"fmt"
	"log"
	"math/rand"
	"os"
	"time"

	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
	"sync"
)

// 奖品类型，枚举 iota 从0开始自增
const (
	giftTypeCoin      = iota // 虚拟币
	giftTypeCoupon           // 优惠券，不相同的编码
	giftTypeCouponFix        // 优惠券，相同的编码
	giftTypeRealSmall        // 实物小奖
	giftTypeRealLarge        // 实物大奖
)

// 最大号码
const rateMax = 10000

// 奖品信息
type gift struct {
	id       int      // 奖品ID
	name     string   // 奖品名称
	pic      string   // 照片链接
	link     string   // 链接
	gtype    int      // 奖品类型
	data     string   // 奖品的数据（特定的配置信息，如：虚拟币面值，固定优惠券的编码）
	datalist []string // 奖品数据集合（特定的配置信息，如：不同的优惠券的编码）
	total    int      // 总数，0 不限量
	left     int      // 剩余数
	inuse    bool     // 是否使用中
	rate     int      // 中奖概率，万分之N,0-10000
	rateMin  int      // 大于等于，中奖的最小号码,0-10000
	rateMax  int      // 小于，中奖的最大号码,0-10000
}

// 文件日志
var logger *log.Logger

// 奖品列表
var giftlist []*gift
var mu sync.Mutex

func main() {
	app := newApp()

	// http://localhost:8080
	app.Run(iris.Addr(":8080"))
}

// 初始化奖品列表信息（管理后台来维护）
func initGift() {
	giftlist = make([]*gift, 5)
	// 1 实物大奖
	g1 := gift{
		id:      1,
		name:    "手机N7",
		pic:     "",
		link:    "",
		gtype:   giftTypeRealLarge,
		data:    "",
		total:   1000,
		left:    1000,
		inuse:   true,
		rate:    10000,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[0] = &g1
	// 2 实物小奖
	g2 := gift{
		id:      2,
		name:    "安全充电 黑色",
		pic:     "",
		link:    "",
		gtype:   giftTypeRealSmall,
		data:    "",
		total:   5,
		left:    5,
		inuse:   true,
		rate:    100,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[1] = &g2
	// 3 虚拟券，相同的编码
	g3 := gift{
		id:      3,
		name:    "商城满2000元减50元优惠券",
		pic:     "",
		link:    "",
		gtype:   giftTypeCouponFix,
		data:    "mall-coupon-2018",
		total:   5,
		left:    5,
		rate:    5000,
		inuse:   true,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[2] = &g3
	// 4 虚拟券，不相同的编码
	g4 := gift{
		id:       4,
		name:     "商城无门槛直降50元优惠券",
		pic:      "",
		link:     "",
		gtype:    giftTypeCoupon,
		data:     "",
		datalist: []string{"c01", "c02", "c03", "c04", "c05"},
		total:    5,
		left:     5,
		inuse:    true,
		rate:     2000,
		rateMin:  0,
		rateMax:  0,
	}
	giftlist[3] = &g4
	// 5 虚拟币
	g5 := gift{
		id:      5,
		name:    "社区10个金币",
		pic:     "",
		link:    "",
		gtype:   giftTypeCoin,
		data:    "10",
		total:   5,
		left:    5,
		inuse:   true,
		rate:    5000,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[4] = &g5

	// 整理奖品数据，把rateMin,rateMax根据rate进行编排
	rateStart := 0
	for _, data := range giftlist {
		if !data.inuse {
			continue
		}
		data.rateMin = rateStart
		data.rateMax = data.rateMin + data.rate
		if data.rateMax >= rateMax {
			// 号码达到最大值，分配的范围重头再来
			data.rateMax = rateMax
			rateStart = 0
		} else {
			rateStart += data.rate
		}
	}
	fmt.Printf("giftlist=%v\n", giftlist)
}

// 初始化日志信息
func initLog() {
	f, _ := os.Create("/var/log/lottery_demo.log")
	logger = log.New(f, "", log.Ldate|log.Lmicroseconds)
}

func newApp() *iris.Application {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&lotteryController{})
	// 初始化日志信息
	initLog()
	// 初始化奖品信息
	initGift()
	return app
}

// 抽奖的控制器
type lotteryController struct {
	Ctx iris.Context
}

// GET http://localhost:8080/
func (c *lotteryController) Get() string {
	count := 0
	total := 0
	for _, data := range giftlist {
		if data.inuse && (data.total == 0 ||
			(data.total > 0 && data.left > 0)) {
			count++
			total += data.left
		}
	}
	return fmt.Sprintf("当前有效奖品种类数量: %d，限量奖品总数量=%d\n", count, total)
}

// GET http://localhost:8080/lucky
func (c *lotteryController) GetLucky() map[string]interface{} {
	mu.Lock()
	defer mu.Unlock()

	code := luckyCode()
	ok := false
	result := make(map[string]interface{})
	result["success"] = ok
	for _, data := range giftlist {
		if !data.inuse || (data.total > 0 && data.left <= 0) {
			continue
		}
		if data.rateMin <= int(code) && data.rateMax > int(code) {
			// 中奖了，抽奖编码在奖品中奖编码范围内
			sendData := ""
			switch data.gtype {
			case giftTypeCoin:
				ok, sendData = sendCoin(data)
			case giftTypeCoupon:
				ok, sendData = sendCoupon(data)
			case giftTypeCouponFix:
				ok, sendData = sendCouponFix(data)
			case giftTypeRealSmall:
				ok, sendData = sendRealSmall(data)
			case giftTypeRealLarge:
				ok, sendData = sendRealLarge(data)
			}
			if ok {
				// 中奖后，成功得到奖品（发奖成功）
				// 生成中奖纪录
				saveLuckyData(code, data.id, data.name, data.link, sendData, data.left)
				result["success"] = ok
				result["id"] = data.id
				result["name"] = data.name
				result["link"] = data.link
				result["data"] = sendData
				break
			}
		}
	}

	return result
}

// 抽奖编码
func luckyCode() int32 {
	seed := time.Now().UnixNano()                                 // rand内部运算的随机数
	code := rand.New(rand.NewSource(seed)).Int31n(int32(rateMax)) // rand计算得到的随机数
	return code
}

// 发奖，虚拟币
func sendCoin(data *gift) (bool, string) {
	if data.total == 0 {
		// 数量无限
		return true, data.data
	} else if data.left > 0 {
		// 还有剩余
		data.left = data.left - 1
		return true, data.data
	} else {
		return false, "奖品已发完"
	}
}

// 发奖，优惠券（不同值）
func sendCoupon(data *gift) (bool, string) {
	if data.left > 0 {
		// 还有剩余的奖品
		left := data.left - 1
		data.left = left
		return true, data.datalist[left]
	} else {
		return false, "奖品已发完"
	}
}

// 发奖，优惠券（固定值）
func sendCouponFix(data *gift) (bool, string) {
	if data.total == 0 {
		// 数量无限
		return true, data.data
	} else if data.left > 0 {
		data.left = data.left - 1
		return true, data.data
	} else {
		return false, "奖品已发完"
	}
}

// 发奖，实物小
func sendRealSmall(data *gift) (bool, string) {
	if data.total == 0 {
		// 数量无限
		return true, data.data
	} else if data.left > 0 {
		data.left = data.left - 1
		return true, data.data
	} else {
		return false, "奖品已发完"
	}
}

// 发奖，实物大
func sendRealLarge(data *gift) (bool, string) {
	if data.total == 0 {
		// 数量无限
		return true, data.data
	} else if data.left > 0 {
		data.left--
		return true, data.data
	} else {
		return false, "奖品已发完"
	}
}

// 记录用户的获奖记录
func saveLuckyData(code int32, id int, name, link, sendData string, left int) {
	logger.Printf("lucky, code=%d, gift=%d, name=%s, link=%s, data=%s, left=%d ", code, id, name, link, sendData, left)
}
```



## 支付宝集五福-alipayFu

```go
/**
 * 支付宝五福
 * 五福的概率来自识别后的参数(AI图片识别MaBaBa)
 * 基础功能：
 * /lucky 只有一个抽奖的接口，奖品信息都是预先配置好的
 * 测试方法：
 * curl "http://localhost:8080/?rate=4,3,2,1,0"
 * curl "http://localhost:8080/lucky?uid=1&rate=4,3,2,1,0"
 * 压力测试：（这里不存在线程安全问题）
 * wrk -t10 -c10 -d 10 "http://localhost:8080/lucky?uid=1&rate=4,3,2,1,0"
 */

package main

import (
	"fmt"
	"log"
	"math/rand"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
)

// 最大号码
const rateMax = 10

// 奖品信息
type gift struct {
	id      int    // 奖品ID
	name    string // 奖品名称
	pic     string // 照片链接
	link    string // 链接
	inuse   bool   // 是否使用中
	rate    int    // 中奖概率，十分之N,0-9
	rateMin int    // 大于等于，中奖的最小号码,0-10
	rateMax int    // 小于，中奖的最大号码,0-10
}

// 文件日志
var logger *log.Logger

func main() {
	app := newApp()

	// http://localhost:8080/
	app.Run(iris.Addr(":8080"))
}

// 初始化奖品列表信息（管理后台来维护）
func newGift() *[5]gift {
	giftlist := new([5]gift)
	// 1 实物大奖
	g1 := gift{
		id:      1,
		name:    "富强福",
		pic:     "富强福.jpg",
		link:    "",
		inuse:   true,
		rate:    4,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[0] = g1
	// 2 实物小奖
	g2 := gift{
		id:      2,
		name:    "和谐福",
		pic:     "和谐福.jpg",
		link:    "",
		inuse:   true,
		rate:    3,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[1] = g2
	// 3 虚拟券，相同的编码
	g3 := gift{
		id:      3,
		name:    "友善福",
		pic:     "友善福.jpg",
		link:    "",
		inuse:   true,
		rate:    2,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[2] = g3
	// 4 虚拟券，不相同的编码
	g4 := gift{
		id:      4,
		name:    "爱国福",
		pic:     "爱国福.jpg",
		link:    "",
		inuse:   true,
		rate:    1,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[3] = g4
	// 5 虚拟币
	g5 := gift{
		id:      5,
		name:    "敬业福",
		pic:     "敬业福.jpg",
		link:    "",
		inuse:   true,
		rate:    0,
		rateMin: 0,
		rateMax: 0,
	}
	giftlist[4] = g5
	return giftlist
}

// 根据概率，计算好的奖品信息列表
func giftRate(rate string) *[5]gift {
	giftlist := newGift()
	rates := strings.Split(rate, ",")
	ratesLen := len(rates)
	// 整理奖品数据，把rateMin,rateMax根据rate进行编排
	rateStart := 0
	for i, data := range giftlist {
		if !data.inuse {
			continue
		}
		grate := 0
		if i < ratesLen { // 避免数组越界
			grate, _ = strconv.Atoi(rates[i])
		}
		giftlist[i].rate = grate
		giftlist[i].rateMin = rateStart
		giftlist[i].rateMax = rateStart + grate
		if giftlist[i].rateMax >= rateMax {
			// 号码达到最大值，分配的范围重头再来
			giftlist[i].rateMax = rateMax
			rateStart = 0
		} else {
			rateStart += grate
		}
	}
	fmt.Printf("giftlist=%v\n", giftlist)
	return giftlist
}

// 初始化日志信息
func initLog() {
	f, _ := os.Create("/var/log/lottery_demo.log")
	logger = log.New(f, "", log.Ldate|log.Lmicroseconds)
}

func newApp() *iris.Application {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&lotteryController{})
	// 初始化日志信息
	initLog()
	return app
}

// 抽奖的控制器
type lotteryController struct {
	Ctx iris.Context
}

// GET http://localhost:8080/?rate=4,3,2,1,0
func (c *lotteryController) Get() string {
	rate := c.Ctx.URLParamDefault("rate", "4,3,2,1,0")
	giftlist := giftRate(rate)
	return fmt.Sprintf("%v\n", giftlist)
}

// GET http://localhost:8080/lucky?uid=1&rate=4,3,2,1,0
func (c *lotteryController) GetLucky() map[string]interface{} {
	uid, _ := c.Ctx.URLParamInt("uid")
	rate := c.Ctx.URLParamDefault("rate", "4,3,2,1,0")
	code := luckyCode()
	ok := false
	result := make(map[string]interface{})
	result["success"] = ok
	giftlist := giftRate(rate)
	for _, data := range giftlist {
		if !data.inuse {
			continue
		}
		if data.rateMin <= int(code) && data.rateMax >= int(code) {
			// 中奖了，抽奖编码在奖品中奖编码范围内
			ok = true
			sendData := data.pic

			if ok {
				// 中奖后，成功得到奖品（发奖成功）
				// 生成中奖纪录
				saveLuckyData(uid, code, data.id, data.name, data.link, sendData)
				result["success"] = ok
				result["uid"] = uid
				result["id"] = data.id
				result["name"] = data.name
				result["link"] = data.link
				result["data"] = sendData
				break
			}
		}
	}

	return result
}

// 抽奖编码
func luckyCode() int32 {
	seed := time.Now().UnixNano()                                 // rand内部运算的随机数
	code := rand.New(rand.NewSource(seed)).Int31n(int32(rateMax)) // rand计算得到的随机数
	return code
}

// 记录用户的获奖记录
func saveLuckyData(uid int, code int32, id int, name, link, sendData string) {
	logger.Printf("lucky, uid=%d, code=%d, gift=%d, name=%s, link=%s, data=%s ", uid, code, id, name, link, sendData)
}
```

## 微博抽奖-weiboRedPacket

- 多个用户发出来多个红包，更多用户来抢红包
- 红包的集合，红包内红包的数量的读写都存在并发安全性问题

- 优化，将红包集合进行散列，减小单个集合的大小

### 实现发红包

```go
/**
 * 微博抢红包
 * 两个步骤
 * 1 抢红包，设置红包总金额，红包个数，返回抢红包的地址
 * curl "http://localhost:8080/set?uid=1&money=100&num=100"
 * 2 抢红包，先到先得，随机得到红包金额
 * curl "http://localhost:8080/get?id=1&uid=1"
 * 注意：
 * 线程不安全1，红包列表 packageList map 的并发读写会产生异常
 * 测试方法： wrk -t10 -c10 -d5  "http://localhost:8080/set?uid=1&money=100&num=10"
 * fatal error: concurrent map writes
 * 线程不安全2，红包里面的金额切片 packageList map[uint32][]uint 并发读写不安全，虽然不会报错
 */
package main

import (
	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
	"os"
	"log"
	"fmt"
	"math/rand"
	"time"
)
// 文件日志
var logger *log.Logger
// 当前有效红包列表，int64是红包唯一ID，[]uint是红包里面随机分到的金额（单位分）
var packageList map[uint32][]uint = make(map[uint32][]uint)

func main() {
	app := newApp()
	app.Run(iris.Addr(":8080"))
}

// 初始化Application
func newApp() *iris.Application {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&lotteryController{})
	return app
}

// 初始化日志
func initLog() {
	f, _ := os.Create("/var/logs/lottery_demo.log")
	logger = log.New(f, "", log.Ldate|log.Lmicroseconds)
}

// 抽奖的控制器
type lotteryController struct {
	Ctx iris.Context
}

// 返回全部红包地址
// GET http://localhost:8080/
func (c *lotteryController) Get() map[uint32][2]int {
	rs := make(map[uint32][2]int)
	for id, list := range packageList {
		var money int
		for _, v := range list {
			money += int(v)
		}
		rs[id] = [2]int{len(list),money}
	}
	return rs
}

// 发红包
// GET http://localhost:8080/set?uid=1&money=100&num=100
func (c *lotteryController) GetSet() string {
	uid, errUid := c.Ctx.URLParamInt("uid")
	money, errMoney := c.Ctx.URLParamFloat64("money")
	num, errNum := c.Ctx.URLParamInt("num")
	if errUid != nil || errMoney != nil || errNum != nil {
		return fmt.Sprintf("参数格式异常，errUid=%s, errMoney=%s, errNum=%s\n", errUid, errMoney, errNum)
	}
	moneyTotal := int(money * 100)
	if uid < 1 || moneyTotal < num || num < 1 {
		return fmt.Sprintf("参数数值异常，uid=%d, money=%d, num=%d\n", uid, money, num)
	}
	// 金额分配算法
	leftMoney := moneyTotal
	leftNum := num
	// 分配的随机数
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	rMax := 0.55 // 随机分配最大比例
	list := make([]uint, num)
	// 大循环开始，只要还有没分配的名额，继续分配
	for leftNum > 0 {
		if leftNum == 1 {
			// 最后一个名额，把剩余的全部给它
			list[num-1] = uint(leftMoney)
			break
		}
		// 剩下的最多只能分配到1分钱时，不用再随机
		if leftMoney == leftNum {
			for i:=num-leftNum; i < num ; i++ {
				list[i] = 1
			}
			break
		}
		// 每次对剩余金额的1%-55%随机，最小1，最大就是剩余金额55%（需要给剩余的名额留下1分钱的生存空间）
		rMoney := int(float64(leftMoney-leftNum) * rMax)
		m := r.Intn(rMoney)
		if m < 1 {
			m = 1
		}
		list[num-leftNum] = uint(m)
		leftMoney -= m
		leftNum--
	}
	// 最后再来一个红包的唯一ID
	id := r.Uint32()
	packageList[id] = list
	// 返回抢红包的URL
	return fmt.Sprintf("/get?id=%d&uid=%d&num=%d\n", id, uid, num)
}

// 抢红包
// GET http://localhost:8080/get?id=1&uid=1
func (c *lotteryController) GetGet() string {
	uid, errUid := c.Ctx.URLParamInt("uid")
	id, errId := c.Ctx.URLParamInt("id")
	if errUid != nil || errId != nil {
		return fmt.Sprintf("参数格式异常，errUid=%s, errId=%s\n", errUid, errId)
	}
	if uid < 1 || id < 1 {
		return fmt.Sprintf("参数数值异常，uid=%d, id=%d\n", uid, id)
	}
	list, ok := packageList[uint32(id)]
	if !ok || len(list) < 1 {
		return fmt.Sprintf("红包不存在,id=%d\n", id)
	}
	// 分配的随机数
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	// 从红包金额中随机得到一个
	i := r.Intn(len(list))
	money := list[i]
	// 更新红包列表中的信息
	if len(list) > 1 {
		if i == len(list) - 1 {
			packageList[uint32(id)] = list[:i]
		} else if i == 0 {
			packageList[uint32(id)] = list[1:]
		} else {
			packageList[uint32(id)] = append(list[:i], list[i+1:]...)
		}
	} else {
		delete(packageList, uint32(id))
	}
	return fmt.Sprintf("恭喜你抢到一个红包，金额为:%d\n", money)
}

```



>需要并发读写时，一般的做法是加锁，但这样性能并不高，Go语言在 1.9 版本中提供了一种效率较高的并发安全的 sync.Map，sync.Map 和 map 不同，不是以语言原生形态提供，而是在 sync 包下的特殊结构。
>
>sync.Map 有以下特性：
>
>- 无须初始化，直接声明即可。
>- sync.Map 不能使用 map 的方式进行取值和设置等操作，而是使用 sync.Map 的方法进行调用，Store 表示存储，Load 表示获取，Delete 表示删除。
>- 使用 Range 配合一个回调函数进行遍历操作，通过回调函数返回内部遍历出来的值，Range 参数中回调函数的返回值在需要继续迭代遍历时，返回 true，终止迭代遍历时，返回 false。



### 使用sync.Map 和chan的方式解决并发安全问题

```go
/**
 * 微博抢红包
 * 两个步骤
 * 1 抢红包，设置红包总金额，红包个数，返回抢红包的地址
 * GET /set?uid=1&money=100&num=100
 * 2 抢红包，先到先得，随机得到红包金额
 * GET /get?id=1&uid=1
 * 注意：
 * 线程安全1，红包列表 packageList map 改用线程安全的 sync.Map
 * 线程安全2，红包里面的金额切片 packageList map[uint32][]uint 并发读写不安全，虽然不会报错
 * 改进：使用 channel 来更新
 */
package main

import (
	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
	"os"
	"log"
	"fmt"
	"math/rand"
	"time"
	"sync"
)

// 文件日志
var logger *log.Logger
// 当前有效红包列表，int64是红包唯一ID，[]uint是红包里面随机分到的金额（单位分）
//var packageList map[uint32][]uint = make(map[uint32][]uint)
var packageList *sync.Map = new(sync.Map)
var chTasks chan task = make(chan task)

func main() {
	app := newApp()
	app.Run(iris.Addr(":8080"))
}

// 初始化Application
func newApp() *iris.Application {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&lotteryController{})

	initLog()
	go fetchPackageMoney()
	return app
}

// 初始化日志
func initLog() {
	f, _ := os.Create("/var/log/lottery_demo.log")
	logger = log.New(f, "", log.Ldate|log.Lmicroseconds)
}

// 单线程死循环，专注处理各个红包中金额切片的数据更新（移除指定位置的金额）
func fetchPackageMoney() {
	for {
		t := <-chTasks
		// 分配的随机数
		r := rand.New(rand.NewSource(time.Now().UnixNano()))
		id := t.id
		l, ok := packageList.Load(id)
		if ok && l != nil {
			list := l.([]uint)
			// 从红包金额中随机得到一个
			i := r.Intn(len(list))
			money := list[i]
			//if i == len(list) - 1 {
			//	packageList[uint32(id)] = list[:i]
			//} else if i == 0 {
			//	packageList[uint32(id)] = list[1:]
			//} else {
			//	packageList[uint32(id)] = append(list[:i], list[i+1:]...)
			//}
			if len(list) > 1 {
				if i == len(list) - 1 {
					packageList.Store(uint32(id), list[:i])
				} else if i == 0 {
					packageList.Store(uint32(id), list[1:])
				} else {
					packageList.Store(uint32(id), append(list[:i], list[i+1:]...))
				}
			} else {
				//delete(packageList, uint32(id))
				packageList.Delete(uint32(id))
			}
			// 回调channel返回
			t.callback <- money
		} else {
			t.callback <- 0
		}
	}
}
// 任务结构
type task struct {
	id uint32
	callback chan uint
}

// 抽奖的控制器
type lotteryController struct {
	Ctx iris.Context
}

// 返回全部红包地址
// GET http://localhost:8080/
func (c *lotteryController) Get() map[uint32][2]int {
	rs := make(map[uint32][2]int)
	//for id, list := range packageList {
	//	var money int
	//	for _, v := range list {
	//		money += int(v)
	//	}
	//	rs[id] = [2]int{len(list),money}
	//}
	packageList.Range(func(key, value interface{}) bool {
		id := key.(uint32)
		list := value.([]uint)
		var money int
		for _, v := range list {
			money += int(v)
		}
		rs[id] = [2]int{len(list),money}
		return true
	})
	return rs
}

// 发红包
// GET http://localhost:8080/set?uid=1&money=100&num=100
func (c *lotteryController) GetSet() string {
	uid, errUid := c.Ctx.URLParamInt("uid")
	money, errMoney := c.Ctx.URLParamFloat64("money")
	num, errNum := c.Ctx.URLParamInt("num")
	if errUid != nil || errMoney != nil || errNum != nil {
		return fmt.Sprintf("参数格式异常，errUid=%s, errMoney=%s, errNum=%s\n", errUid, errMoney, errNum)
	}
	moneyTotal := int(money * 100)
	if uid < 1 || moneyTotal < num || num < 1 {
		return fmt.Sprintf("参数数值异常，uid=%d, money=%d, num=%d\n", uid, money, num)
	}
	// 金额分配算法
	leftMoney := moneyTotal
	leftNum := num
	// 分配的随机数
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	// 随机分配最大比例
	rMax := 0.55
	if num >= 1000 {
		rMax = 0.01
	}else if num >= 100 {
		rMax = 0.1
	} else if num >= 10 {
		rMax = 0.3
	}
	list := make([]uint, num)
	// 大循环开始，只要还有没分配的名额，继续分配
	for leftNum > 0 {
		if leftNum == 1 {
			// 最后一个名额，把剩余的全部给它
			list[num-1] = uint(leftMoney)
			break
		}
		// 剩下的最多只能分配到1分钱时，不用再随机
		if leftMoney == leftNum {
			for i:=num-leftNum; i < num ; i++ {
				list[i] = 1
			}
			break
		}
		// 每次对剩余金额的1%-55%随机，最小1，最大就是剩余金额55%（需要给剩余的名额留下1分钱的生存空间）
		rMoney := int(float64(leftMoney-leftNum) * rMax)
		m := r.Intn(rMoney)
		if m < 1 {
			m = 1
		}
		list[num-leftNum] = uint(m)
		leftMoney -= m
		leftNum--
	}
	// 最后再来一个红包的唯一ID
	id := r.Uint32()
	//packageList[id] = list
	packageList.Store(id, list)
	// 返回抢红包的URL
	return fmt.Sprintf("/get?id=%d&uid=%d&num=%d\n", id, uid, num)
}

// 抢红包
// GET http://localhost:8080/get?id=1&uid=1
func (c *lotteryController) GetGet() string {
	uid, errUid := c.Ctx.URLParamInt("uid")
	id, errId := c.Ctx.URLParamInt("id")
	if errUid != nil || errId != nil {
		return fmt.Sprintf("参数格式异常，errUid=%s, errId=%s\n", errUid, errId)
	}
	if uid < 1 || id < 1 {
		return fmt.Sprintf("参数数值异常，uid=%d, id=%d\n", uid, id)
	}
	//list, ok := packageList[uint32(id)]
	l, ok := packageList.Load(uint32(id))
	if !ok {
		return fmt.Sprintf("红包不存在,id=%d\n", id)
	}
	list := l.([]uint)
	if len(list) < 1 {
		return fmt.Sprintf("红包不存在,id=%d\n", id)
	}
	// 更新红包列表中的信息（移除这个金额），构造一个任务
	callback := make(chan uint)
	t := task{id: uint32(id), callback: callback}
	// 把任务发送给channel
	chTasks <- t
	// 回调的channel等待处理结果
	money := <- callback
	if money <= 0 {
		fmt.Println(uid, "很遗憾，没能抢到红包")
		return fmt.Sprintf("很遗憾，没能抢到红包\n")
	} else {
		fmt.Println(uid, "抢到一个红包，金额为:", money)
		logger.Printf("weiboReadPacket success uid=%d, id=%d, money=%d\n", uid, id, money)
		return fmt.Sprintf("恭喜你抢到一个红包，金额为:%d\n", money)
	}
}

```



### 再次压测验证和优化改造

```go
/**
 * 微博抢红包
 * 两个步骤
 * 1 抢红包，设置红包总金额，红包个数，返回抢红包的地址
 * GET /set?uid=1&money=100&num=100
 * 2 抢红包，先到先得，随机得到红包金额
 * GET /get?id=1&uid=1
 * 注意：
 * 线程安全1，红包列表 packageList map 改用线程安全的 sync.Map
 * 线程安全2，红包里面的金额切片 packageList map[uint32][]uint 并发读写不安全，虽然不会报错
 * 优化 channel 的吞吐量，启动多个处理协程来执行 channel 的消费
 */
package main

import (
	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
	"os"
	"log"
	"fmt"
	"math/rand"
	"time"
	"sync"
)

// 文件日志
var logger *log.Logger
// 当前有效红包列表，int64是红包唯一ID，[]uint是红包里面随机分到的金额（单位分）
//var packageList map[uint32][]uint = make(map[uint32][]uint)
var packageList *sync.Map = new(sync.Map)
//var chTasks chan task = make(chan task)
const taskNum = 16
var chTaskList []chan task = make([]chan task, taskNum)

func main() {
	app := newApp()
	app.Run(iris.Addr(":8080"))
}

// 初始化Application
func newApp() *iris.Application {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&lotteryController{})

	initLog()
	for i:=0; i<taskNum; i++ {
		chTaskList[i] = make(chan task)
		go fetchPackageMoney(chTaskList[i])
	}
	return app
}

// 初始化日志
func initLog() {
	f, _ := os.Create("/var/log/lottery_demo.log")
	logger = log.New(f, "", log.Ldate|log.Lmicroseconds)
}

// 单线程死循环，专注处理各个红包中金额切片的数据更新（移除指定位置的金额）
func fetchPackageMoney(chTasks chan task) {
	for {
		t := <-chTasks
		// 分配的随机数
		r := rand.New(rand.NewSource(time.Now().UnixNano()))
		id := t.id
		l, ok := packageList.Load(id)
		if ok && l != nil {
			list := l.([]uint)
			// 从红包金额中随机得到一个
			i := r.Intn(len(list))
			money := list[i]
			//if i == len(list) - 1 {
			//	packageList[uint32(id)] = list[:i]
			//} else if i == 0 {
			//	packageList[uint32(id)] = list[1:]
			//} else {
			//	packageList[uint32(id)] = append(list[:i], list[i+1:]...)
			//}
			if len(list) > 1 {
				if i == len(list) - 1 {
					packageList.Store(uint32(id), list[:i])
				} else if i == 0 {
					packageList.Store(uint32(id), list[1:])
				} else {
					packageList.Store(uint32(id), append(list[:i], list[i+1:]...))
				}
			} else {
				//delete(packageList, uint32(id))
				packageList.Delete(uint32(id))
			}
			// 回调channel返回
			t.callback <- money
		} else {
			t.callback <- 0
		}
	}
}
// 任务结构
type task struct {
	id uint32
	callback chan uint
}

// 抽奖的控制器
type lotteryController struct {
	Ctx iris.Context
}

// 返回全部红包地址
// GET http://localhost:8080/
func (c *lotteryController) Get() map[uint32][2]int {
	rs := make(map[uint32][2]int)
	//for id, list := range packageList {
	//	var money int
	//	for _, v := range list {
	//		money += int(v)
	//	}
	//	rs[id] = [2]int{len(list),money}
	//}
	packageList.Range(func(key, value interface{}) bool {
		id := key.(uint32)
		list := value.([]uint)
		var money int
		for _, v := range list {
			money += int(v)
		}
		rs[id] = [2]int{len(list),money}
		return true
	})
	return rs
}

// 发红包
// GET http://localhost:8080/set?uid=1&money=100&num=100
func (c *lotteryController) GetSet() string {
	uid, errUid := c.Ctx.URLParamInt("uid")
	money, errMoney := c.Ctx.URLParamFloat64("money")
	num, errNum := c.Ctx.URLParamInt("num")
	if errUid != nil || errMoney != nil || errNum != nil {
		return fmt.Sprintf("参数格式异常，errUid=%s, errMoney=%s, errNum=%s\n", errUid, errMoney, errNum)
	}
	moneyTotal := int(money * 100)
	if uid < 1 || moneyTotal < num || num < 1 {
		return fmt.Sprintf("参数数值异常，uid=%d, money=%d, num=%d\n", uid, money, num)
	}
	// 金额分配算法
	leftMoney := moneyTotal
	leftNum := num
	// 分配的随机数
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	// 随机分配最大比例
	rMax := 0.55
	if num >= 1000 {
		rMax = 0.01
	}else if num >= 100 {
		rMax = 0.1
	} else if num >= 10 {
		rMax = 0.3
	}
	list := make([]uint, num)
	// 大循环开始，只要还有没分配的名额，继续分配
	for leftNum > 0 {
		if leftNum == 1 {
			// 最后一个名额，把剩余的全部给它
			list[num-1] = uint(leftMoney)
			break
		}
		// 剩下的最多只能分配到1分钱时，不用再随机
		if leftMoney == leftNum {
			for i:=num-leftNum; i < num ; i++ {
				list[i] = 1
			}
			break
		}
		// 每次对剩余金额的1%-55%随机，最小1，最大就是剩余金额55%（需要给剩余的名额留下1分钱的生存空间）
		rMoney := int(float64(leftMoney-leftNum) * rMax)
		m := r.Intn(rMoney)
		if m < 1 {
			m = 1
		}
		list[num-leftNum] = uint(m)
		leftMoney -= m
		leftNum--
	}
	// 最后再来一个红包的唯一ID
	id := r.Uint32()
	//packageList[id] = list
	packageList.Store(id, list)
	// 返回抢红包的URL
	return fmt.Sprintf("/get?id=%d&uid=%d&num=%d\n", id, uid, num)
}

// 抢红包
// GET http://localhost:8080/get?id=1&uid=1
func (c *lotteryController) GetGet() string {
	uid, errUid := c.Ctx.URLParamInt("uid")
	id, errId := c.Ctx.URLParamInt("id")
	if errUid != nil || errId != nil {
		return fmt.Sprintf("参数格式异常，errUid=%s, errId=%s\n", errUid, errId)
	}
	if uid < 1 || id < 1 {
		return fmt.Sprintf("参数数值异常，uid=%d, id=%d\n", uid, id)
	}
	//list, ok := packageList[uint32(id)]
	l, ok := packageList.Load(uint32(id))
	if !ok {
		return fmt.Sprintf("红包不存在,id=%d\n", id)
	}
	list := l.([]uint)
	if len(list) < 1 {
		return fmt.Sprintf("红包不存在,id=%d\n", id)
	}
	// 更新红包列表中的信息（移除这个金额），构造一个任务
	callback := make(chan uint)
	t := task{id: uint32(id), callback: callback}
	// 把任务发送给channel
	chTasks := chTaskList[id % taskNum]
	chTasks <- t
	// 回调的channel等待处理结果
	money := <- callback
	if money <= 0 {
		fmt.Println(uid, "很遗憾，没能抢到红包")
		return fmt.Sprintf("很遗憾，没能抢到红包\n")
	} else {
		fmt.Println(uid, "抢到一个红包，金额为:", money)
		logger.Printf("weiboReadPacket success uid=%d, id=%d, money=%d\n", uid, id, money)
		return fmt.Sprintf("恭喜你抢到一个红包，金额为:%d\n", money)
	}
}
```





## 抽奖大转盘-wheel



```go
/**
 * 大转盘程序
 * curl http://localhost:8080/
 * curl http://localhost:8080/debug
 * curl http://localhost:8080/prize
 * 固定几个奖品，不同的中奖概率或者总数量限制
 * 每一次转动抽奖，后端计算出这次抽奖的中奖情况，并返回对应的奖品信息
 *
 * 线程不安全，因为获奖概率低，并发更新库存的冲突很少能出现，不容易发现线程安全性问题
 * 压力测试：
 * wrk -t10 -c100 -d5 "http://localhost:8080/prize"
 */
package main

import (
	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
	"fmt"
	"strings"
	"time"
	"math/rand"
	)

// 奖品中奖概率
type Prate struct {
	Rate int		// 万分之N的中奖概率
	Total int		// 总数量限制，0 表示无限数量
	CodeA int		// 中奖概率起始编码（包含）
	CodeB int		// 中奖概率终止编码（包含）
	Left int 		// 剩余数
}
// 奖品列表
var prizeList []string = []string{
	"一等奖，火星单程船票",
	"二等奖，凉飕飕南极之旅",
	"三等奖，iPhone一部",
	"",							// 没有中奖
}
// 奖品的中奖概率设置，与上面的 prizeList 对应的设置
var rateList []Prate = []Prate{
	Prate{1, 1, 0, 0, 1},
	Prate{2, 2, 1, 2, 2},
	Prate{5, 10, 3, 5, 10},
	Prate{100,0, 0, 9999, 0},
}

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

// GET http://localhost:8080/
func (c *lotteryController) Get() string {
	c.Ctx.Header("Content-Type", "text/html")
	return fmt.Sprintf("大转盘奖品列表：<br/> %s", strings.Join(prizeList, "<br/>\n"))
}

// GET http://localhost:8080/prize
func (c *lotteryController) GetPrize() string {
	c.Ctx.Header("Content-Type", "text/html")

	// 第一步，抽奖，根据随机数匹配奖品
	seed := time.Now().UnixNano()
	r := rand.New(rand.NewSource(seed))
	// 得到个人的抽奖编码
	code := r.Intn(10000)
	//fmt.Println("GetPrize code=", code)
	var myprize string
	var prizeRate *Prate
	// 从奖品列表中匹配，是否中奖
	for i, prize := range prizeList {
		rate := &rateList[i]
		if code >= rate.CodeA && code <= rate.CodeB {
			// 满足中奖条件
			myprize = prize
			prizeRate = rate
			break
		}
	}
	if myprize == "" {
		// 没有中奖
		myprize = "很遗憾，再来一次"
		return myprize
	}
	// 第二步，发奖，是否可以发奖
	if prizeRate.Total == 0 {
		// 无限奖品
		fmt.Println("中奖： ", myprize)
		return myprize
	} else if prizeRate.Left > 0 {
		// 还有剩余奖品
		prizeRate.Left -= 1
		fmt.Println("中奖： ", myprize)
		return myprize
	} else {
		// 有限且没有剩余奖品，无法发奖
		myprize = "很遗憾，再来一次"
		return myprize
	}
}

// GET http://localhost:8080/debug
func (c *lotteryController) GetDebug() string {
	c.Ctx.Header("Content-Type", "text/html")
	return fmt.Sprintf("获奖概率： %v", rateList)
}
```



### sync.Mutex和atomic改造性能对比

```go
/**
 * 大转盘程序
 * curl http://localhost:8080/
 * curl http://localhost:8080/debug
 * curl http://localhost:8080/prize
 * 固定几个奖品，不同的中奖概率或者总数量限制
 * 每一次转动抽奖，后端计算出这次抽奖的中奖情况，并返回对应的奖品信息
 *
 * 增加互斥锁，保证并发库存更新的正常
 * 压力测试：
 * wrk -t10 -c100 -d5 "http://localhost:8080/prize"
 */
package main

import (
	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
	"fmt"
	"strings"
	"time"
	"math/rand"
	"sync"
)

// 奖品中奖概率
type Prate struct {
	Rate int		// 万分之N的中奖概率
	Total int		// 总数量限制，0 表示无限数量
	CodeA int		// 中奖概率起始编码（包含）
	CodeB int		// 中奖概率终止编码（包含）
	Left int 		// 剩余数
}
// 奖品列表
var prizeList []string = []string{
	"一等奖，火星单程船票",
	"二等奖，凉飕飕南极之旅",
	"三等奖，iPhone一部",
	"",							// 没有中奖
}
// 奖品的中奖概率设置，与上面的 prizeList 对应的设置
var rateList []Prate = []Prate{
	//Prate{1, 1, 0, 0, 1},
	//Prate{2, 2, 1, 2, 2},
	Prate{5, 1000, 0, 9999, 1000},
	//Prate{100,0, 0, 9999, 0},
}

var mu sync.Mutex

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

// GET http://localhost:8080/
func (c *lotteryController) Get() string {
	c.Ctx.Header("Content-Type", "text/html")
	return fmt.Sprintf("大转盘奖品列表：<br/> %s", strings.Join(prizeList, "<br/>\n"))
}

// GET http://localhost:8080/prize
func (c *lotteryController) GetPrize() string {
	c.Ctx.Header("Content-Type", "text/html")
	mu.Lock()
	defer mu.Unlock()

	// 第一步，抽奖，根据随机数匹配奖品
	seed := time.Now().UnixNano()
	r := rand.New(rand.NewSource(seed))
	// 得到个人的抽奖编码
	code := r.Intn(10000)
	//fmt.Println("GetPrize code=", code)
	var myprize string
	var prizeRate *Prate
	// 从奖品列表中匹配，是否中奖
	for i, prize := range prizeList {
		rate := &rateList[i]
		if code >= rate.CodeA && code <= rate.CodeB {
			// 满足中奖条件
			myprize = prize
			prizeRate = rate
			break
		}
	}
	if myprize == "" {
		// 没有中奖
		myprize = "很遗憾，再来一次"
		return myprize
	}
	// 第二步，发奖，是否可以发奖
	if prizeRate.Total == 0 {
		// 无限奖品
		fmt.Println("中奖： ", myprize)
		return myprize
	} else if prizeRate.Left > 0 {
		// 还有剩余奖品
		prizeRate.Left -= 1
		fmt.Println("中奖： ", myprize)
		return myprize
	} else {
		// 有限且没有剩余奖品，无法发奖
		myprize = "很遗憾，再来一次"
		return myprize
	}
}

// GET http://localhost:8080/debug
func (c *lotteryController) GetDebug() string {
	c.Ctx.Header("Content-Type", "text/html")
	return fmt.Sprintf("获奖概率： %v", rateList)
}

```

### safe 2

```go
/**
 * 大转盘程序
 * curl http://localhost:8080/
 * curl http://localhost:8080/debug
 * curl http://localhost:8080/prize
 * 固定几个奖品，不同的中奖概率或者总数量限制
 * 每一次转动抽奖，后端计算出这次抽奖的中奖情况，并返回对应的奖品信息
 *
 * 不用互斥锁，而是用CAS操作来更新，保证并发库存更新的正常
 * 压力测试：
 * wrk -t10 -c100 -d5 "http://localhost:8080/prize"
 */
package main

import (
	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
	"fmt"
	"strings"
	"time"
	"math/rand"
	"sync/atomic"
)


// 奖品中奖概率
type Prate struct {
	Rate int		// 万分之N的中奖概率
	Total int		// 总数量限制，0 表示无限数量
	CodeA int		// 中奖概率起始编码（包含）
	CodeB int		// 中奖概率终止编码（包含）
	Left *int32 		// 剩余数
}
// 奖品列表
var prizeList []string = []string{
	"一等奖，火星单程船票",
	"二等奖，凉飕飕南极之旅",
	"三等奖，iPhone一部",
	"",							// 没有中奖
}
var giftLeft = int32(1000)
// 奖品的中奖概率设置，与上面的 prizeList 对应的设置
var rateList []Prate = []Prate{
	//Prate{1, 1, 0, 0, 1},
	//Prate{2, 2, 1, 2, 2},
	Prate{5, 1000, 0, 9999, &giftLeft},
	//Prate{100,0, 0, 9999, 0},
}

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

// GET http://localhost:8080/
func (c *lotteryController) Get() string {
	c.Ctx.Header("Content-Type", "text/html")
	return fmt.Sprintf("大转盘奖品列表：<br/> %s", strings.Join(prizeList, "<br/>\n"))
}

// GET http://localhost:8080/prize
func (c *lotteryController) GetPrize() string {
	c.Ctx.Header("Content-Type", "text/html")
	// 第一步，抽奖，根据随机数匹配奖品
	seed := time.Now().UnixNano()
	r := rand.New(rand.NewSource(seed))
	// 得到个人的抽奖编码
	code := r.Intn(10000)
	//fmt.Println("GetPrize code=", code)
	var myprize string
	var prizeRate *Prate
	// 从奖品列表中匹配，是否中奖
	for i, prize := range prizeList {
		rate := &rateList[i]
		if code >= rate.CodeA && code <= rate.CodeB {
			// 满足中奖条件
			myprize = prize
			prizeRate = rate
			break
		}
	}
	if myprize == "" {
		// 没有中奖
		myprize = "很遗憾，再来一次"
		return myprize
	}
	// 第二步，发奖，是否可以发奖
	if prizeRate.Total == 0 {
		// 无限奖品
		fmt.Println("中奖： ", myprize)
		return myprize
	} else if *prizeRate.Left > 0 {
		// 还有剩余奖品，使用 CAS 操作来做安全更新
		left := atomic.AddInt32(prizeRate.Left, -1)
		if left >= 0 {
			fmt.Println("中奖： ", myprize)
			return myprize
		}
	}
	// 有限且没有剩余奖品，无法发奖
	myprize = "很遗憾，再来一次"
	return myprize
}

// GET http://localhost:8080/debug
func (c *lotteryController) GetDebug() string {
	c.Ctx.Header("Content-Type", "text/html")
	return fmt.Sprintf("获奖概率： %v", rateList)
}

```

## 六种抽奖活动总结

- 关键点，奖品类型，数量有限，中奖概率，发奖规则
- 并发安全性问题，互斥，队列，CAS递减
- 优化，通过散列减少单个集合的大小



