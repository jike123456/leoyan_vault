- 上电初始化 GPIO 和 USART1
- 启动后打印 shell 提示
- 主循环里按字符接收串口命令
- 回车后做 `Trim_Command()` 再交给 `APP_Shell_ProcessCommand()`
- 按键中断里直接控制 LED

