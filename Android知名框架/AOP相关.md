### AOP 面向切面技术

| APT（注解处理器） | 编译期生成新代码，不修改已有类   | APT / KAPT / KSP            |
| ----------------- | -------------------------------- | --------------------------- |
| 插桩入口          | Gradle 构建过程中的 hook 点      | Transform / Instrumentation |
| 字节码修改工具    | 真正用于操作 .class 字节码的工具 | Javassist / ASM/ ByteX      |

