## Vim
**用较少的学习成本，换来较大的效率提升**。
## 学习资源
[Missing Semester - 3. Vim 编辑器](https://missing-semester-cn.github.io/2020/editors/)
## 术语说明
vim 中有一些术语：
- mode：标准模式 normal、插入模式 insert、可视模式 visual 等。
- verb：vim 里执行的操作，比如删除 `d`、修改 `c`、拷贝 `y`、查找 `f` 等。verb 后面需要跟一个 motion，表示该操作生效的范围。
- motion：vim 里移动的范围，比如向右一个字母 `l`、向下一行 `j`、向右一个单词 `w` 等。本文中有时候也称其为 “range”。
## Modal editing  模态编辑
- 标准模式（Normal Mode）：进入 vim 的默认模式，这个模式下按下任何键不会实际输入到文本中，按下 `:` 可以执行命令
- 插入模式（Insert Mode）：在标准模式按下 `i` 进入插入模式，此时可以输入文本；按下 `<ESC>` 退出插入模式(建议配置 `jj` 退出插入模式)
