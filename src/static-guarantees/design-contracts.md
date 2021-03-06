# 设计合约

在上一章中，我们编写了一个接口,但是该接口**没有**严格执行设计合约。让我们再看一下我们假想的GPIO配置寄存器：

|名字    |位号         | 值    |含义    |注意事项 |
| ---:  |------------:| ----:|------: | ----: |
|启用| 0 | 0 | 禁用| 禁用GPIO |
|   |   | 1 |启用 |启用GPIO |
|方向| 1 | 0 |输入|将方向设置为输入|
|   |    | 1 |输出|将方向设置为输出|
| 输入模式 | 2..3 | 00 | 高电阻 |将输入设置为高阻|
|         |      | 01 |拉低|输入引脚被拉低|
|         |      | 10 |拉高|输入引脚被拉高|
|         |      | 11 |状态无效|不设|
| 输出模式 | 4    | 0 |低|引脚被驱动为低电平|
|         |      | 1 |高|输出引脚被驱动为高电平
| 输入状态 | 5    | x |输入值|如果输入<1.5v，则为0；如果输入> = 1.5v，则为1 |


如果我们改为在运行时检查是否遵循了设计合约,即在使用底层硬件之前检查状态，则我们可能会编写如下所示的代码：



```rust , ignore
/// GPIO interface
struct GpioConfig {
    /// GPIO Configuration structure generated by svd2rust
    periph: GPIO_CONFIG,
}

impl GpioConfig {
    pub fn set_enable(&mut self, is_enabled: bool) {
        self.periph.modify(|_r, w| {
            w.enable().set_bit(is_enabled)
        });
    }

    pub fn set_direction(&mut self, is_output: bool) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Must be enabled to set direction
            return Err(());
        }

        self.periph.modify(|r, w| {
            w.direction().set_bit(is_output)
        });

        Ok(())
    }

    pub fn set_input_mode(&mut self, variant: InputMode) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Must be enabled to set input mode
            return Err(());
        }

        if self.periph.read().direction().bit_is_set() {
            // Direction must be input
            return Err(());
        }

        self.periph.modify(|_r, w| {
            w.input_mode().variant(variant)
        });

        Ok(())
    }

    pub fn set_output_status(&mut self, is_high: bool) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Must be enabled to set output status
            return Err(());
        }

        if self.periph.read().direction().bit_is_clear() {
            // Direction must be output
            return Err(());
        }

        self.periph.modify(|_r, w| {
            w.output_mode.set_bit(is_high)
        });

        Ok(())
    }

    pub fn get_input_status(&self) -> Result<bool, ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Must be enabled to get status
            return Err(());
        }

        if self.periph.read().direction().bit_is_set() {
            // Direction must be input
            return Err(());
        }

        Ok(self.periph.read().input_status().bit_is_set())
    }
}
```

因为我们需要遵循硬件的访问限制，所以最终需要进行大量的运行时检查，这一方面浪费了时间和资源，另一方面开发人员也会不太满意(这样的代码写起来乏味,用这个代码的人也容易出错)。

## 类型状态机

如果我们使用Rust的类型系统执行状态转换规则会是什么样子呢？举个例子：

```rust , ignore
/// GPIO interface
struct GpioConfig<ENABLED, DIRECTION, MODE> {
    /// GPIO Configuration structure generated by svd2rust
    periph: GPIO_CONFIG,
    enabled: ENABLED,
    direction: DIRECTION,
    mode: MODE,
}

// Type states for MODE in GpioConfig
struct Disabled;
struct Enabled;
struct Output;
struct Input;
struct PulledLow;
struct PulledHigh;
struct HighZ;
struct DontCare;

/// These functions may be used on any GPIO Pin
impl<EN, DIR, IN_MODE> GpioConfig<EN, DIR, IN_MODE> {
    pub fn into_disabled(self) -> GpioConfig<Disabled, DontCare, DontCare> {
        self.periph.modify(|_r, w| w.enable.disabled());
        GpioConfig {
            periph: self.periph,
            enabled: Disabled,
            direction: DontCare,
            mode: DontCare,
        }
    }

    pub fn into_enabled_input(self) -> GpioConfig<Enabled, Input, HighZ> {
        self.periph.modify(|_r, w| {
            w.enable.enabled()
             .direction.input()
             .input_mode.high_z()
        });
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: HighZ,
        }
    }

    pub fn into_enabled_output(self) -> GpioConfig<Enabled, Output, DontCare> {
        self.periph.modify(|_r, w| {
            w.enable.enabled()
             .direction.output()
             .input_mode.set_high()
        });
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Output,
            mode: DontCare,
        }
    }
}

/// This function may be used on an Output Pin
impl GpioConfig<Enabled, Output, DontCare> {
    pub fn set_bit(&mut self, set_high: bool) {
        self.periph.modify(|_r, w| w.output_mode.set_bit(set_high));
    }
}

/// These methods may be used on any enabled input GPIO
impl<IN_MODE> GpioConfig<Enabled, Input, IN_MODE> {
    pub fn bit_is_set(&self) -> bool {
        self.periph.read().input_status.bit_is_set()
    }

    pub fn into_input_high_z(self) -> GpioConfig<Enabled, Input, HighZ> {
        self.periph.modify(|_r, w| w.input_mode().high_z());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: HighZ,
        }
    }

    pub fn into_input_pull_down(self) -> GpioConfig<Enabled, Input, PulledLow> {
        self.periph.modify(|_r, w| w.input_mode().pull_low());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: PulledLow,
        }
    }

    pub fn into_input_pull_up(self) -> GpioConfig<Enabled, Input, PulledHigh> {
        self.periph.modify(|_r, w| w.input_mode().pull_high());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: PulledHigh,
        }
    }
}
```

现在，让我们看一下使用它的代码是什么样的：

```rust , ignore
/*
 * Example 1: Unconfigured to High-Z input
 */
let pin: GpioConfig<Disabled, _, _> = get_gpio();

// Can't do this, pin isn't enabled!
// pin.into_input_pull_down();

// Now turn the pin from unconfigured to a high-z input
let input_pin = pin.into_enabled_input();

// Read from the pin
let pin_state = input_pin.bit_is_set();

// Can't do this, input pins don't have this interface!
// input_pin.set_bit(true);

/*
 * Example 2: High-Z input to Pulled Low input
 */
let pulled_low = input_pin.into_input_pull_down();
let pin_state = pulled_low.bit_is_set();

/*
 * Example 3: Pulled Low input to Output, set high
 */
let output_pin = pulled_low.into_enabled_output();
output_pin.set_bit(true);

// Can't do this, output pins don't have this interface!
// output_pin.into_input_pull_down();
```

这绝对是存储引脚状态的便捷方法，但是为什么要这样做呢？为什么这比将状态作为`enum`存储在我们的 `GpioConfig`结构中更好？

## 编译时功能安全

因为我们在编译时完全强制执行设计约束，所以不会产生运行时成本。当引脚处于输入模式时，无法设置输出模式。相反，您必须通过将其转换为输出引脚。因此由于不必在执行功能之前检查当前状态，不会造成运行时间损失。

同样，由于这些状态是由类型系统强制执行的，因此该接口的使用者不可能错误地使用。如果他们尝试执行非法的状态转换，则代码将无法编译！