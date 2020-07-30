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







## 抽奖大转盘-wheel





