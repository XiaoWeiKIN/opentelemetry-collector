# configoptional

[![PkgGoDev](https://pkg.go.dev/badge/go.opentelemetry.io/collector/config/configoptional)](https://pkg.go.dev/go.opentelemetry.io/collector/config/configoptional)

## 概述

`configoptional` 包提供了一个泛型的 `Optional[T]` 类型，用于表示可能存在或不存在的配置值。这个包是 OpenTelemetry Collector 配置系统的一部分，用于优雅地处理可选配置字段。

## 核心概念

`Optional[T]` 类型支持三种状态（flavor）：

1. **None** - 无值状态
   - 表示配置值不存在
   - 零值的 `Optional[T]` 即为 `None`
   - 反序列化时如果配置为 nil，保持 None 状态

2. **Some(value)** - 有值状态
   - 包含一个具体的值
   - 通过 `Some()` 构造函数创建
   - 反序列化时会填充配置值

3. **Default(defaultVal)** - 默认值状态（仅支持结构体类型）
   - 包含一个默认值，用于配置反序列化
   - 如果配置为 nil，使用默认值
   - 如果配置有值，则覆盖默认值

## 主要功能

### 构造函数

```go
// 创建一个包含值的 Optional
opt := configoptional.Some(MyStruct{Field: "value"})

// 创建一个包含默认值的 Optional（仅支持结构体）
opt := configoptional.Default(MyStruct{Field: "default"})

// 创建一个空的 Optional
opt := configoptional.None[MyStruct]()
```

### 方法

- `HasValue() bool` - 检查 Optional 是否有值
- `Get() *T` - 获取值的指针，如果无值则返回 nil
- `GetOrInsertDefault() *T` - 获取值或插入默认值（将 None 转为 Some(零值)，Default 转为 Some(默认值)）

### 配置反序列化

`Optional[T]` 实现了 `confmap.Unmarshaler` 接口，支持从配置文件反序列化：

```go
type Config struct {
    OptionalField Optional[SubConfig] `mapstructure:"optional_field"`
}
```

### 特性门控：enabled 字段

通过 `configoptional.AddEnabledField` 特性门控（从 v0.138.0 开始），可以在配置中使用 `enabled` 字段来显式启用或禁用可选配置：

```yaml
optional_field:
  enabled: true  # 显式启用
  other_field: value

another_optional:
  enabled: false  # 显式禁用，即使有其他配置值也会被忽略
  other_field: ignored
```

### 验证

`Optional[T]` 实现了 `xconfmap.Validator` 接口：
- None 和 Default 状态不会触发验证
- Some 状态会验证实际的值

## 限制

1. 类型 T 必须可以解引用为结构体类型（用于反序列化/序列化）
2. 类型 T 不能包含带有 `mapstructure:"enabled"` 标签的字段（这个名称被保留用于特性门控功能）
3. 标量值不支持反序列化操作

## 使用示例

### 基本用法

```go
type Config struct {
    Required    string            `mapstructure:"required"`
    OptionalSub Optional[SubCfg]  `mapstructure:"optional_sub"`
}

type SubCfg struct {
    Value string `mapstructure:"value"`
}

// 设置默认配置
cfg := Config{
    Required:    "",
    OptionalSub: Default(SubCfg{Value: "default"}),
}

// 从配置文件反序列化
conf := confmap.NewFromStringMap(map[string]any{
    "required": "must_have",
    // optional_sub 未提供，将使用默认值
})
conf.Unmarshal(&cfg)

// 检查和使用可选值
if cfg.OptionalSub.HasValue() {
    value := cfg.OptionalSub.Get()
    fmt.Println(value.Value)
}
```

### 程序化使用

```go
// 创建一个空的可选值
opt := None[MyStruct]()

// 获取或插入默认值（零值）
myStruct := opt.GetOrInsertDefault()
myStruct.Field = "value"

// 现在 opt 是 Some 状态，包含修改后的值
```

## 设计理念

`Optional[T]` 的设计目标是提供类似于指针的行为，但具有更好的默认值支持：
- None 类似于 nil 指针
- Some 类似于非 nil 指针
- Default 提供了指针无法提供的默认值功能

在序列化和反序列化方面，`Optional[T]` 尽可能地模拟指针的行为，使得它可以作为指针的替代品使用。

## 依赖

- `go.opentelemetry.io/collector/confmap` - 配置映射支持
- `go.opentelemetry.io/collector/confmap/xconfmap` - 扩展配置映射功能
- `go.opentelemetry.io/collector/featuregate` - 特性门控支持

## 许可证

Apache License 2.0