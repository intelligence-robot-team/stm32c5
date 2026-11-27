---
name: embedded-c-oop
description: 嵌入式 C 语言 OOP 与五层软件架构规范，使用结构体组合、Ops 函数指针表、container_of 实现多态与向下转型，使用 board_init 手动完成板级设备初始化。
---

# 嵌入式 C 语言 OOP 架构 Skill

## 1. 五层软件架构

采用 APP / MODULE / DEVICE / PLATFORM / BSP 五层架构。

| 层次 | 职责 |
|---|---|
| APP | 应用业务逻辑、任务调度 |
| MODULE | 多设备组合、算法、功能集成 |
| DEVICE | 设备抽象、基类、子类、Ops 多态 |
| PLATFORM | 芯片硬件操作接口、平台适配 |
| BSP | 板级硬件映射、设备实例创建与初始化 |

依赖关系：

```text
APP
 ↓
MODULE
 ↓
DEVICE
 ↓
PLATFORM
 ↓
硬件
```

BSP 负责具体开发板的配置和初始化，向上提供已经初始化好的设备句柄。

核心原则：

- APP 不关心设备实现。
- MODULE 不关心具体硬件。
- DEVICE 不直接操作寄存器或依赖具体 MCU HAL。
- PLATFORM 屏蔽芯片差异。
- BSP 屏蔽开发板差异。

## 2. C 语言 OOP 实现

采用结构体组合模拟继承，Ops 函数指针表实现多态。

统一以电机为例。

### 2.1 基类 Motor

```c
typedef struct Motor Motor;

typedef struct
{
    void (*SetSpeed)(Motor *self, int32_t speed);
    void (*Stop)(Motor *self);
} MotorOps;

struct Motor
{
    const MotorOps *ops;
    int32_t speed;
};
```

基类负责公共成员和统一接口。

### 2.2 子类 Motor_CAN / Motor_PWM

```c
typedef struct
{
    Motor base;
    uint8_t can_id;
} Motor_CAN;

typedef struct
{
    Motor base;
    uint32_t pwm_channel;
} Motor_PWM;
```

子类通过包含基类结构体模拟继承，并保存自己的硬件参数。

### 2.3 Ops 表实现多态

```c
static const MotorOps can_ops = {
    .SetSpeed = CAN_SetSpeed,
    .Stop = CAN_Stop
};

static const MotorOps pwm_ops = {
    .SetSpeed = PWM_SetSpeed,
    .Stop = PWM_Stop
};
```

子类初始化时，将基类的 Ops 指针指向对应实现：

```c
void Motor_CAN_Init(Motor_CAN *self, uint8_t id)
{
    self->can_id = id;
    self->base.ops = &can_ops;
}

void Motor_PWM_Init(Motor_PWM *self, uint32_t channel)
{
    self->pwm_channel = channel;
    self->base.ops = &pwm_ops;
}
```

基类统一接口：

```c
void Motor_SetSpeed(Motor *self, int32_t speed)
{
    if (self && self->ops && self->ops->SetSpeed)
        self->ops->SetSpeed(self, speed);
}
```

上层调用：

```c
Motor_SetSpeed(g_motor, 1000);
```

根据对象初始化时绑定的 Ops 表，自动调用 CAN 或 PWM 的具体实现。

### 2.4 container_of 向下转型

当子类实现需要访问自己的私有成员时，从基类指针还原子类指针。

```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member))
```

CAN 子类实现：

```c
static void CAN_SetSpeed(Motor *self, int32_t speed)
{
    Motor_CAN *motor =
        container_of(self, Motor_CAN, base);

    motor->base.speed = speed;

    Platform_CAN_SendSpeed(motor->can_id, speed);
}
```

注意：只能对真实子类对象中的对应基类成员进行转换，保证类型匹配和对象生命周期有效。

## 3. PLATFORM 与 BSP

### 3.1 PLATFORM：轻量化

只提供统一的底层硬件操作接口，屏蔽 MCU 和 HAL 差异。

```text
platform/
├── platform_can.h
├── platform_can.c
├── platform_pwm.h
└── platform_pwm.c
```

例如：

```c
void Platform_CAN_SendSpeed(uint8_t id, int32_t speed);

void Platform_PWM_SetDuty(uint32_t channel, float duty);
```

具体实现调用对应 MCU 的 HAL 或寄存器接口。

更换 MCU 时，主要修改 PLATFORM 实现。

### 3.2 BSP：轻量化

只保留板级配置、设备实例和初始化入口。

```text
bsp/
├── board_config.h
├── board_init.h
└── board_init.c
```

主要职责：

1. 创建具体设备对象。
2. 指定设备硬件资源。
3. 调用 DEVICE 初始化函数。
4. 将子类对象向上转型为基类指针。
5. 对外提供统一设备句柄。

示例：

```c
// board_init.c

static Motor_CAN motor_left;
static Motor_PWM motor_right;

Motor *g_motor_left;
Motor *g_motor_right;

void Board_Init(void)
{
    Motor_CAN_Init(&motor_left, 0x01);
    Motor_PWM_Init(&motor_right, 1);

    g_motor_left = &motor_left.base;
    g_motor_right = &motor_right.base;
}
```

应用层只需要：

```c
Motor_SetSpeed(g_motor_left, 1000);
Motor_SetSpeed(g_motor_right, 1000);
```

更换电机类型时，只修改 BSP 中的对象类型、初始化参数和绑定关系，上层不变。

## 4. 初始化机制

采用手动注册，不采用 section + initcall 自动注册。

main 统一调用：

```c
int main(void)
{
    HAL_Init();
    SystemClock_Config();

    Board_Init();
    Module_Init();
    App_Init();

    while (1)
    {
        App_Task();
    }
}
```

初始化职责：

- main：系统启动流程。
- Board_Init：板级硬件和设备对象初始化。
- Module_Init：功能模块初始化。
- App_Init：应用初始化。

底层外设依赖必须先准备好，再初始化对应 DEVICE。

board_init 是板级组合入口，不负责实现具体设备驱动。

## 5. 推荐工程目录

```text
Project/
├── app/                 # 应用层
│   ├── main.c
│   └── Chassis_task.c
│
├── module/              # 功能集成层
│   ├── chassis_contrl/
│   └── 
│
├── device/              # 设备抽象层
│   ├── motor/
│   │   ├── motor.h
│   │   ├── motor.c
│   │   ├── motor_can.c
│   └── imu/
│   │   ├── imu.h
│   │   ├── imu.c
│   │   ├── imu_zdyz.c
│
├── platform/            # 平台适配层
│
└── bsp/                 # 板级支持层
    ├── board_config.h
    ├── board_init.h
    └── board_init.c
```

不强制每个模块都拆成多个文件，按实际复杂度组织。

## 6. AI 代码生成规则

AI 在生成代码时必须遵守：

1. 严格遵循五层架构及各层职责。
2. 使用 C 语言结构体组合实现继承。
3. 使用 Ops 函数指针表实现多态。
4. 子类初始化时绑定对应 Ops 表。
5. 使用 container_of 实现合法向下转型。
6. DEVICE 不直接调用具体 MCU HAL。
7. PLATFORM 负责硬件操作抽象。
8. BSP 负责设备实例、硬件映射、初始化和句柄绑定。
9. main 调用 Board_Init，不直接初始化大量具体设备。
10. 使用手动初始化，不默认使用 section + initcall。
11. 简单驱动不强制套用 OOP。
12. 避免过度抽象、重复封装和无意义的文件拆分。

新增设备时，依次考虑：

- 定义统一基类和公共接口。
- 根据需要实现具体子类。
- 定义 Ops 表。
- 实现子类初始化和向下转型。
- 通过 PLATFORM 操作底层硬件。
- 在 BSP 中创建实例并绑定统一句柄。

---

**最终设计思想：**

基类定义接口，子类实现功能，Ops 实现多态，container_of 实现向下转型。

DEVICE 隔离设备实现，PLATFORM 隔离芯片差异，BSP 隔离开发板差异。

board_init 负责将具体硬件对象组织起来，向上提供统一接口。

APP 和 MODULE 不需要知道底层使用什么电机、什么 MCU、什么引脚。
---