---
author: LinuxROS
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg4NzY1MDE5NQ==&mid=2247486170&idx=1&sn=2ea14ca95e24f03fe1f7de3135128c35&chksm=ce901093522c7147825486ddc3e2cab34d09b1ddceb1ed77976eb25d655693b7e409a9dc583d&mpshare=1&scene=1&srcid=0724y8MvKFZQhmQ5l4wzOjNK&sharer_shareinfo=7d7b0ff16f3e58cc1da5996fff20a853&sharer_shareinfo_first=7d7b0ff16f3e58cc1da5996fff20a853#rd
saved: 2026-07-24 15:03:28
tags:
  - 笔记同步助手
id: 7b9dcdbd-3365-426e-a959-4a90fb1c9023
---

公众号名称：LinuxROS

作者名称：LinuxROS

发布时间：2026-06-17 08:00

> 用C语言写面向对象代码，很多人觉得不可能。没有Class、没有继承、没有多态，怎么搞？其实一个struct加上函数指针就够了。这篇文章把GoF 23种设计模式全部用纯C实现一遍，**每段代码已在Ubuntu 24.04 + GCC环境下编译运行验证通过**。不管你是写嵌入式固件还是搞Linux，这些套路都能用上。

---

## 一、C语言面向对象三板斧

在C语言里模拟面向对象，核心就三招：

| OOP特性 | C语言实现 | 干什么用 |
| --- | --- | --- |
| 封装 | `struct`\+ 函数指针 | 数据和方法打包 |
| 继承 | 结构体嵌套 | 子类首成员放父类 |
| 多态 | 函数指针数组 | 模拟vtable |
| 生命周期 | static / malloc | 控制对象创建销毁 |

记住这个套路，后面23种模式全靠它。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/15e53709b0eb9e8989ad0348253d7b80_MD5.jpg]]

## 二、创建型模式：5种对象创建套路

创建型模式解决"怎么创建对象"的问题，把new的过程封装起来，调用方不用关心对象怎么来的。

### 2.1 单例模式：全局只许有一个

嵌入式开发里全局配置、日志模块、驱动句柄，往往只需要一个实例。用`static`变量就能保证唯一性。

**饿汉式**——程序启动就创建，线程安全：

```
#include <stdio.h>

typedef struct Singleton {
    void (*show)(void);
} Singleton;

static Singleton g_instance;

void Singleton_Show(void) {
    printf("饿汉单例：全局唯一实例\n");
}

Singleton* GetSingletonInstance(void) {
    static int init_flag = 0;
    if (!init_flag) {
        g_instance.show = Singleton_Show;
        init_flag = 1;
    }
    return &g_instance;
}

int main(void) {
    Singleton *s1 = GetSingletonInstance();
    Singleton *s2 = GetSingletonInstance();
    printf("s1 == s2: %s\n", s1 == s2 ? "YES" : "NO");
    s1->show();
    return 0;
}
```

**懒汉式**——用到才创建，省资源但非线程安全：

```
#include <stdio.h>
#include <stdlib.h>

typedef struct Singleton {
    void (*show)(void);
} Singleton;

static Singleton* g_lazy_instance = NULL;

void LazyShow(void) { printf("懒汉单例\n"); }

Singleton* GetLazyInstance(void) {
    if (g_lazy_instance == NULL) {
        g_lazy_instance = (Singleton*)malloc(sizeof(Singleton));
        g_lazy_instance->show = LazyShow;
    }
    return g_lazy_instance;
}

int main(void) {
    Singleton* s = GetLazyInstance();
    s->show();
    free(g_lazy_instance);
    return 0;
}
```

> 嵌入式场景优先用饿汉式，没有并发问题。多线程环境用懒汉式要加锁，或者用pthread\_once。

### 2.2 工厂模式：创建逻辑别散落各处

对象创建逻辑集中到一个函数里，调用方只传参数，不关心内部怎么new。

**简单工厂**：

```
#include <stdio.h>
#include <string.h>

typedef struct Animal {
    void (*cry)(void);
} Animal;

void DogCry(void)  { printf("小狗：汪汪汪！\n"); }
void CatCry(void)  { printf("小猫：喵喵喵！\n"); }

Animal* CreateAnimal(const char* type) {
    static Animal dog = {DogCry};
    static Animal cat = {CatCry};
    if (strcmp(type, "dog") == 0) return &dog;
    if (strcmp(type, "cat") == 0) return &cat;
    return NULL;
}

int main(void) {
    Animal* a1 = CreateAnimal("dog");
    Animal* a2 = CreateAnimal("cat");
    if (a1) a1->cry();
    if (a2) a2->cry();
    return 0;
}
```

**抽象工厂**——产品族场景，比如不同品牌的GUI组件：

```
#include <stdio.h>
#include <stdlib.h>

typedef struct Button { void (*Paint)(void); } Button;
typedef struct TextBox { void (*Paint)(void); } TextBox;

typedef struct GUIFactory {
    Button*  (*CreateButton)(void);
    TextBox* (*CreateTextBox)(void);
} GUIFactory;

/* Windows风格 */
void WinBtnPaint(void)  { printf("Windows按钮\n"); }
void WinTxTPaint(void)  { printf("Windows文本框\n"); }
static Button  winBtn  = {WinBtnPaint};
static TextBox winTxt  = {WinTxTPaint};
Button*  WinCreateButton(void)  { return &winBtn; }
TextBox* WinCreateTextBox(void) { return &winTxt; }

/* Linux风格 */
void LinuxBtnPaint(void)  { printf("Linux按钮\n"); }
void LinuxTxTPaint(void)  { printf("Linux文本框\n"); }
static Button  linuxBtn  = {LinuxBtnPaint};
static TextBox linuxTxt  = {LinuxTxTPaint};
Button*  LinuxCreateButton(void)  { return &linuxBtn; }
TextBox* LinuxCreateTextBox(void) { return &linuxTxt; }

int main(void) {
    GUIFactory winFactory = {WinCreateButton, WinCreateTextBox};
    GUIFactory linuxFactory = {LinuxCreateButton, LinuxCreateTextBox};

    Button* b1 = winFactory.CreateButton();
    b1->Paint();
    Button* b2 = linuxFactory.CreateButton();
    b2->Paint();
    return 0;
}
```

### 2.3 建造者模式：复杂对象分步拼装

当一个对象有很多部件需要按顺序组装时，建造者模式把"拼装步骤"和"具体实现"分开。

```
#include <stdio.h>
#include <string.h>

typedef struct House {
    char walls[50];
    char roof[50];
    char floor[50];
} House;

typedef struct Builder {
    void (*BuildWall)(House*);
    void (*BuildRoof)(House*);
    void (*BuildFloor)(House*);
    House* (*GetHouse)(void);
} Builder;

void BuildWall(House* h)  { strcpy(h->walls, "红砖墙"); printf("建墙：%s\n", h->walls); }
void BuildRoof(House* h)  { strcpy(h->roof, "瓦片顶"); printf("建顶：%s\n", h->roof); }
void BuildFloor(House* h) { strcpy(h->floor, "木地板"); printf("建地：%s\n", h->floor); }

static House g_house;
House* GetHouse(void) { return &g_house; }

void DirectorConstruct(Builder* b, House* h) {
    b->BuildWall(h);
    b->BuildRoof(h);
    b->BuildFloor(h);
}

int main(void) {
    Builder b = {BuildWall, BuildRoof, BuildFloor, GetHouse};
    DirectorConstruct(&b, &g_house);
    House* h = b.GetHouse();
    printf("房子：墙=%s 顶=%s 地=%s\n", h->walls, h->roof, h->floor);
    return 0;
}
```

### 2.4 原型模式：克隆比new快

需要大量相似对象时，克隆一个已有的比从头创建快得多。

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct Prototype {
    char name[30];
    int  value;
    struct Prototype* (*Clone)(struct Prototype*);
} Prototype;

Prototype* ProtoClone(Prototype* self) {
    Prototype* p = (Prototype*)malloc(sizeof(Prototype));
    memcpy(p, self, sizeof(Prototype));
    printf("克隆：%s\n", p->name);
    return p;
}

int main(void) {
    Prototype original = {"原始对象", 42, ProtoClone};
    Prototype* copy = original.Clone(&original);
    copy->value = 99;
    printf("原始：%s value=%d\n", original.name, original.value);
    printf("克隆：%s value=%d\n", copy->name, copy->value);
    free(copy);
    return 0;
}
```

### 创建型模式速查

| 模式 | 一句话 | C语言关键点 |
| --- | --- | --- |
| 单例 | 全局唯一 | static变量+init\_flag |
| 工厂 | 创建逻辑集中 | if-else/函数指针分发 |
| 抽象工厂 | 产品族创建 | 函数指针结构体 |
| 建造者 | 分步拼装 | Director调Builder函数指针 |
| 原型 | 克隆代替创建 | memcpy深拷贝 |

---

## 三、结构型模式：7种组合拼装手法

结构型模式解决"怎么把类和对象组合成更大的结构"，核心思想是组合优于继承。

### 3.1 适配器模式：让不兼容的接口合作

旧模块接口和新系统对不上？套一层适配器就行。

```
#include <stdio.h>

void Old220VPower(void) {
    printf("旧设备：220V 供电\n");
}

typedef struct TargetPower {
    void (*Supply)(void);
} TargetPower;

void AdapterSupply(void) {
    printf("适配器：电压转换中...\n");
    Old220VPower();
}

int main(void) {
    TargetPower t = {AdapterSupply};
    t.Supply();
    return 0;
}
```

### 3.2 桥接模式：两个维度独立变化

图形有形状和颜色两个维度，用桥接模式让它们各自扩展，不用组合爆炸。

```
#include <stdio.h>

typedef struct Color {
    void (*ShowColor)(void);
} Color;

void RedColor(void)   { printf("红色\n"); }
void BlueColor(void)  { printf("蓝色\n"); }

typedef struct Shape {
    Color *color;
    void (*Draw)(struct Shape*);
} Shape;

void DrawShape(Shape* s) {
    printf("绘制图形，颜色：");
    s->color->ShowColor();
}

int main(void) {
    Color red = {RedColor};
    Color blue = {BlueColor};
    Shape s1 = {&red, DrawShape};
    Shape s2 = {&blue, DrawShape};
    s1.Draw(&s1);
    s2.Draw(&s2);
    return 0;
}
```

### 3.3 装饰器模式：比继承更灵活的功能叠加

给咖啡加牛奶、加糖，每加一层就包一层，不用改原来的类。

```
#include <stdio.h>

typedef struct Coffee {
    int (*Cost)(void);
    void (*Info)(void);
} Coffee;

int SimpleCost(void) { return 10; }
void SimpleInfo(void){ printf("基础咖啡 "); }

int MilkCost(void)   { return 15; }
void MilkInfo(void) { printf("加牛奶 "); }

int main(void) {
    Coffee base = {SimpleCost, SimpleInfo};
    Coffee milk = {MilkCost, MilkInfo};

    base.Info();
    printf("价格：%d元\n", base.Cost());

    milk.Info();
    printf("价格：%d元\n", milk.Cost());

    return 0;
}
```

### 3.4 组合模式：树形结构统一接口

文件系统、组织架构、UI组件树——部分和整体用同一个接口操作。

```
#include <stdio.h>
#include <string.h>

typedef struct Component {
    char name[20];
    void (*Show)(struct Component*, int depth);
} Component;

void LeafShow(Component* leaf, int d) {
    for(int i=0; i<d; i++) printf("  ");
    printf("- %s(叶子)\n", leaf->name);
}

void CompositeShow(Component* self, int d) {
    for(int i=0; i<d; i++) printf("  ");
    printf("+ %s(容器)\n", self->name);
}

int main(void) {
    Component root = {"根节点", CompositeShow};
    Component leaf1 = {"叶子1", LeafShow};
    Component leaf2 = {"叶子2", LeafShow};

    root.Show(&root, 0);
    leaf1.Show(&leaf1, 1);
    leaf2.Show(&leaf2, 1);

    return 0;
}
```

### 3.5 外观模式：复杂子系统一个入口搞定

子系统模块太多？套一个Facade，对外只暴露一个简单接口。

```
#include <stdio.h>

void SubSystem1(void) { printf("子系统1：初始化\n"); }
void SubSystem2(void) { printf("子系统2：加载数据\n"); }
void SubSystem3(void) { printf("子系统3：执行业务\n"); }

void FacadeWork(void) {
    printf("外观模式统一入口\n");
    SubSystem1();
    SubSystem2();
    SubSystem3();
}

int main(void) {
    FacadeWork();
    return 0;
}
```

### 3.6 享元模式：对象复用省内存

大量重复对象吃内存？把共享部分抽出来复用，只传不同的外部状态。

```
#include <stdio.h>
#include <string.h>

#define MAX_POOL 5

typedef struct Flyweight {
    char key[10];
    void (*Operate)(void);
} Flyweight;

static Flyweight pool[MAX_POOL];
static int pool_cnt = 0;

void OpA(void) { printf("操作A\n"); }
void OpB(void) { printf("操作B\n"); }

Flyweight* GetFlyweight(const char* key) {
    for(int i=0; i<pool_cnt; i++) {
        if(strcmp(pool[i].key, key) == 0) {
            printf("复用已有：%s\n", key);
            return &pool[i];
        }
    }
    strcpy(pool[pool_cnt].key, key);
    printf("创建新对象：%s\n", key);
    return &pool[pool_cnt++];
}

int main(void) {
    Flyweight* f1 = GetFlyweight("A");
    f1->Operate = OpA; f1->Operate();
    Flyweight* f2 = GetFlyweight("A");
    f2->Operate();
    Flyweight* f3 = GetFlyweight("B");
    f3->Operate = OpB; f3->Operate();
    return 0;
}
```

### 3.7 代理模式：访问对象前先过我这关

权限校验、延迟加载、远程访问——在真实对象前面挡一层代理。

```
#include <stdio.h>

typedef struct Subject {
    void (*Request)(void);
} Subject;

void RealRequest(void) {
    printf("真实业务执行\n");
}

void ProxyRequest(void) {
    printf("代理前置：权限校验\n");
    RealRequest();
    printf("代理后置：日志记录\n");
}

int main(void) {
    Subject proxy = {ProxyRequest};
    proxy.Request();
    return 0;
}
```

### 结构型模式速查

| 模式 | 一句话 | 典型场景 |
| --- | --- | --- |
| 适配器 | 接口转换 | 旧模块对接新系统 |
| 桥接 | 两个维度解耦 | 形状×颜色、设备×驱动 |
| 装饰器 | 功能叠加 | 中间件链、过滤器链 |
| 组合 | 树形统一接口 | 文件系统、UI组件树 |
| 外观 | 简化入口 | SDK封装、子系统聚合 |
| 享元 | 对象复用 | 字符缓存、线程池 |
| 代理 | 访问控制 | 权限校验、延迟加载 |

---

## 四、行为型模式（上）：6种对象通信套路

行为型模式解决对象之间怎么通信、职责怎么分配的问题。

### 4.1 策略模式：消灭if-else分支

算法经常切换？把每种算法封装成函数指针，运行时替换。

```
#include <stdio.h>

typedef struct Strategy {
    int (*Calc)(int, int);
} Strategy;

int AddCalc(int a, int b) { return a + b; }
int SubCalc(int a, int b) { return a - b; }
int MulCalc(int a, int b) { return a * b; }

typedef struct Context {
    Strategy *st;
    int (*DoCalc)(struct Context*, int, int);
} Context;

int DoCalc(Context* c, int a, int b) {
    return c->st->Calc(a, b);
}

int main(void) {
    Strategy add = {AddCalc};
    Strategy sub = {SubCalc};
    Context ctx = {0};
    ctx.DoCalc = DoCalc;

    ctx.st = &add;
    printf("10 + 5 = %d\n", ctx.DoCalc(&ctx, 10, 5));

    ctx.st = ⊂
    printf("10 - 5 = %d\n", ctx.DoCalc(&ctx, 10, 5));

    return 0;
}
```

### 4.2 模板方法模式：骨架固定步骤可变

算法的整体流程不变，但某些步骤由子类决定。C语言里用函数指针实现"钩子"。

```
#include <stdio.h>

typedef struct Template {
    void (*Step1)(void);
    void (*Step2)(void);
    void (*Run)(struct Template*);
} Template;

void TemplateRun(Template* t) {
    printf("流程开始\n");
    t->Step1();
    t->Step2();
    printf("流程结束\n");
}

void Step1A(void) { printf("步骤1A：初始化\n"); }
void Step2A(void) { printf("步骤2A：处理\n"); }

int main(void) {
    Template A = {Step1A, Step2A, TemplateRun};
    A.Run(&A);
    return 0;
}
```

### 4.3 观察者模式：状态变了自动通知

一个对象状态改变，所有关注它的对象自动收到通知。事件驱动架构的基础。

```
#include <stdio.h>
#include <string.h>

#define MAX_OBS 5

typedef struct Observer Observer;
typedef struct Subject {
    char state[20];
    Observer* obs[MAX_OBS];
    int count;
    void (*Attach)(struct Subject*, Observer*);
    void (*Notify)(struct Subject*);
} Subject;

struct Observer {
    char name[20];
    void (*Update)(Observer*, const char*);
};

void SubjectAttach(Subject* s, Observer* o) {
    s->obs[s->count++] = o;
}

void SubjectNotify(Subject* s) {
    printf("发布者：状态 [%s]\n", s->state);
    for(int i=0; i<s->count; i++) {
        s->obs[i]->Update(s->obs[i], s->state);
    }
}

void ObserverUpdate(Observer* o, const char* state) {
    printf("  [%s] 收到：%s\n", o->name, state);
}

int main(void) {
    Subject sub = {"ready", {0}, 0, SubjectAttach, SubjectNotify};
    Observer o1 = {"观察者1", ObserverUpdate};
    Observer o2 = {"观察者2", ObserverUpdate};

    sub.Attach(&sub, &o1);
    sub.Attach(&sub, &o2);
    strcpy(sub.state, "更新了");
    sub.Notify(&sub);
    return 0;
}
```

### 4.4 迭代器模式：遍历逻辑别暴露内部

集合的内部结构不该暴露给外部，用迭代器封装遍历过程。

```
#include <stdio.h>

typedef struct Iterator {
    int* data;
    int  size;
    int  pos;
    int  (*HasNext)(struct Iterator*);
    int  (*Next)(struct Iterator*);
} Iterator;

int IterHasNext(Iterator* it) { return it->pos < it->size; }
int IterNext(Iterator* it)    { return it->data[it->pos++]; }

int main(void) {
    int arr[] = {10, 20, 30, 40, 50};
    Iterator it = {arr, 5, 0, IterHasNext, IterNext};
    while (it.HasNext(&it)) {
        printf("%d ", it.Next(&it));
    }
    printf("\n");
    return 0;
}
```

### 4.5 责任链模式：请求沿着链条传递

多个处理器组成一条链，请求沿链传递直到有人处理。

```
#include <stdio.h>

typedef struct Handler {
    int level;
    void (*Handle)(struct Handler*, int);
    struct Handler* next;
} Handler;

void HandlerFunc(Handler* h, int request) {
    if (request <= h->level) {
        printf("Handler(level=%d) 处理请求：%d\n", h->level, request);
    } else if (h->next) {
        h->next->Handle(h->next, request);
    } else {
        printf("无人处理请求：%d\n", request);
    }
}

int main(void) {
    Handler h3 = {30, HandlerFunc, NULL};
    Handler h2 = {20, HandlerFunc, &h3};
    Handler h1 = {10, HandlerFunc, &h2};

    h1.Handle(&h1, 5);
    h1.Handle(&h1, 15);
    h1.Handle(&h1, 25);
    h1.Handle(&h1, 50);
    return 0;
}
```

### 4.6 命令模式：请求封装成对象

把请求封装成命令对象，实现撤销、队列、日志等功能。

```
#include <stdio.h>

typedef struct Command {
    void (*Execute)(void);
    void (*Undo)(void);
} Command;

void LightOn(void)  { printf("开灯\n"); }
void LightOff(void) { printf("关灯\n"); }

int main(void) {
    Command cmd_on  = {LightOn, LightOff};
    Command cmd_off = {LightOff, LightOn};

    cmd_on.Execute();
    cmd_on.Undo();
    cmd_off.Execute();
    cmd_off.Undo();
    return 0;
}
```

### 行为型模式（上）速查

| 模式 | 一句话 | C语言关键点 |
| --- | --- | --- |
| 策略 | 算法可替换 | 函数指针运行时切换 |
| 模板方法 | 骨架固定步骤可变 | 函数指针做钩子 |
| 观察者 | 一对多通知 | 回调函数数组 |
| 迭代器 | 遍历封装 | pos指针+HasNext/Next |
| 责任链 | 链式传递 | next指针逐级传递 |
| 命令 | 请求封装 | Execute/Undo函数指针 |

---

## 五、行为型模式（下）：5种高级交互模式

### 5.1 备忘录模式：状态快照随时回滚

需要撤销操作？把状态存起来，随时恢复。

```
#include <stdio.h>
#include <string.h>

typedef struct Memento {
    char state[20];
} Memento;

typedef struct Originator {
    char state[20];
    Memento* (*Save)(struct Originator*);
    void (*Restore)(struct Originator*, Memento*);
} Originator;

static Memento saved;

Memento* OriginatorSave(Originator* o) {
    strcpy(saved.state, o->state);
    printf("保存状态：%s\n", saved.state);
    return &saved;
}

void OriginatorRestore(Originator* o, Memento* m) {
    strcpy(o->state, m->state);
    printf("恢复状态：%s\n", o->state);
}

int main(void) {
    Originator o = {"状态A", OriginatorSave, OriginatorRestore};
    Memento* m = o.Save(&o);
    strcpy(o.state, "状态B");
    printf("当前状态：%s\n", o.state);
    o.Restore(&o, m);
    printf("当前状态：%s\n", o.state);
    return 0;
}
```

### 5.2 状态模式：状态机自动切换行为

同一个对象，在不同状态下干不同的事。状态一换，行为跟着变。

```
#include <stdio.h>

typedef struct State {
    void (*Handle)(void);
    const char* name;
} State;

void FreeHandle(void) { printf("空闲状态\n"); }
void BusyHandle(void) { printf("忙碌状态\n"); }
void StopHandle(void) { printf("停止状态\n"); }

int main(void) {
    State freeState = {FreeHandle, "空闲"};
    State busyState = {BusyHandle, "忙碌"};

    State* current = &freeState;
    current->Handle();

    current = &busyState;
    current->Handle();

    return 0;
}
```

### 5.3 访问者模式：数据结构和操作分离

数据结构不变，但操作经常新增。访问者模式把操作从数据结构里抽出来。

```
#include <stdio.h>

typedef struct Element Element;

typedef struct Visitor {
    void (*VisitA)(void);
    void (*VisitB)(void);
} Visitor;

struct Element {
    void (*Accept)(Visitor*);
};

void ElementAAccept(Visitor* v) { v->VisitA(); }
void ElementBAccept(Visitor* v) { v->VisitB(); }

void Visitor1A(void) { printf("访问者1 处理元素A\n"); }
void Visitor1B(void) { printf("访问者1 处理元素B\n"); }
void Visitor2A(void) { printf("访问者2 导出元素A\n"); }
void Visitor2B(void) { printf("访问者2 导出元素B\n"); }

int main(void) {
    Element a = {ElementAAccept};
    Element b = {ElementBAccept};
    Visitor v1 = {Visitor1A, Visitor1B};
    Visitor v2 = {Visitor2A, Visitor2B};

    a.Accept(&v1);
    b.Accept(&v1);
    a.Accept(&v2);
    b.Accept(&v2);
    return 0;
}
```

### 5.4 中介者模式：网状通信变星型

多个对象互相通信太复杂？引入中介者，所有对象只跟中介者通信。

```
#include <stdio.h>

typedef struct Mediator Mediator;
typedef struct Colleague {
    const char* name;
    Mediator* mediator;
    void (*Send)(struct Colleague*, const char*);
} Colleague;

struct Mediator {
    void (*Notify)(Mediator*, Colleague*, const char*);
};

void ColleagueSend(Colleague* c, const char* msg) {
    printf("[%s] 发送：%s\n", c->name, msg);
    c->mediator->Notify(c->mediator, c, msg);
}

void MediatorNotify(Mediator* m, Colleague* from, const char* msg) {
    (void)m;
    printf("[中介者] 转发 from [%s]：%s\n", from->name, msg);
}

int main(void) {
    Mediator m = {MediatorNotify};
    Colleague c1 = {"同事A", &m, ColleagueSend};
    Colleague c2 = {"同事B", &m, ColleagueSend};
    c1.Send(&c1, "你好B");
    c2.Send(&c2, "你好A");
    return 0;
}
```

### 5.5 解释器模式：简单文法解析

给定一个文法，定义一个解释器来解释句子。嵌入式里的命令解析、配置文件解析可以用这个思路。

```
#include <stdio.h>
#include <string.h>

typedef struct Context {
    char input[50];
    int  pos;
} Context;

typedef struct Expression {
    int (*Interpret)(struct Context*);
} Expression;

int NumberInterpret(struct Context* ctx) {
    int val = 0;
    while (ctx->pos < (int)strlen(ctx->input) && ctx->input[ctx->pos] >= '0' && ctx->input[ctx->pos] <= '9') {
        val = val * 10 + (ctx->input[ctx->pos] - '0');
        ctx->pos++;
    }
    return val;
}

int PlusInterpret(struct Context* ctx) {
    int left = NumberInterpret(ctx);
    while (ctx->pos < (int)strlen(ctx->input) && ctx->input[ctx->pos] == '+') {
        ctx->pos++;
        left += NumberInterpret(ctx);
    }
    return left;
}

int main(void) {
    struct Context ctx = {"12+34+5", 0};
    Expression expr = {PlusInterpret};
    int result = expr.Interpret(&ctx);
    printf("12+34+5 = %d\n", result);
    return 0;
}
```

### 行为型模式（下）速查

| 模式 | 一句话 | C语言关键点 |
| --- | --- | --- |
| 备忘录 | 状态快照回滚 | Memento结构体存状态 |
| 状态 | 行为随状态变 | 函数指针切换 |
| 访问者 | 操作与数据分离 | Accept+Visit双重分派 |
| 中介者 | 星型通信 | Mediator转发消息 |
| 解释器 | 文法解析 | 递归下降解释 |

---

## 六、23种设计模式总览

### 分类速查表

| 类别 | 数量 | 模式 |
| --- | --- | --- |
| 创建型 | 5 | 单例、工厂、抽象工厂、建造者、原型 |
| 结构型 | 7 | 适配器、桥接、装饰器、组合、外观、享元、代理 |
| 行为型 | 11 | 策略、模板方法、观察者、迭代器、责任链、命令、备忘录、状态、访问者、中介者、解释器 |

### C语言实现核心技巧

| 技巧 | 作用 | 对应OOP特性 |
| --- | --- | --- |
| struct + 函数指针 | 数据和方法打包 | 封装 |
| 结构体嵌套 | 子类包含父类 | 继承 |
| 函数指针数组 | 运行时替换实现 | 多态 |
| static变量 | 全局唯一实例 | 单例 |
| malloc/free | 动态对象生命周期 | 对象创建销毁 |
| 回调函数 | 一对多通知 | 观察者 |
| next指针 | 链式传递 | 责任链 |

### 容易混淆的模式对比

| 易混淆对 | 区别 |
| --- | --- |
| 策略 vs 状态 | 策略由客户端选择，状态自动切换 |
| 代理 vs 装饰器 | 代理控制访问，装饰器增强功能 |
| 外观 vs 中介者 | 外观简化调用，中介者解耦通信 |
| 适配器 vs 装饰器 | 适配器改接口，装饰器加功能 |
| 组合 vs 装饰器 | 组合管树形结构，装饰器管功能叠加 |

---

## 七、常见踩坑与解法

| 坑 | 原因 | 解法 |
| --- | --- | --- |
| 懒汉单例线程不安全 | 多线程同时判断NULL | 用饿汉式或加互斥锁 |
| 享元内外状态混淆 | 概念不清 | 内部状态可共享，外部状态不可 |
| 观察者内存泄漏 | 忘记注销 | 用完调Detach移除 |
| 责任链死循环 | next指针成环 | 检查链尾必须为NULL |
| 装饰器层数过多 | 过度包装 | 控制装饰层数，必要时用策略替代 |
| 原型浅拷贝 | 指针成员未深拷贝 | 实现深拷贝Clone函数 |
| 工厂新增产品改代码 | 违反开闭原则 | 用策略模式或注册表替代 |

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/49866271_1784876606177?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg4NzY1MDE5NQ%3D%3D%26mid%3D2247486170%26idx%3D1%26sn%3D2ea14ca95e24f03fe1f7de3135128c35%26chksm%3Dce901093522c7147825486ddc3e2cab34d09b1ddceb1ed77976eb25d655693b7e409a9dc583d%26mpshare%3D1%26scene%3D1%26srcid%3D0724y8MvKFZQhmQ5l4wzOjNK%26sharer_shareinfo%3D7d7b0ff16f3e58cc1da5996fff20a853%26sharer_shareinfo_first%3D7d7b0ff16f3e58cc1da5996fff20a853%23rd&s=obsidian)